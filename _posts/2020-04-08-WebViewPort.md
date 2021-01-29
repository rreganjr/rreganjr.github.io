---
layout: post
title: Web Browser Viewport
---

### Notes on Web Browser Viewport

from [Viewport concepts](https://developer.mozilla.org/en-US/docs/Web/CSS/Viewport_concepts) and [CSSOM View Module](https://www.w3.org/TR/cssom-view/)

* viewport - the size of the area for viewing the web page in the browser window
* window - the size of the window of the browser including the controls not part of a web page. always >= viewport
  *  `window.innerWidth` is _always_ the width of the browser window viewport including the vertical scrollbar if present or zero if there is no viewport.
  * The area within the `innerHeight` and `innerWidth` is the layout viewport.
  * zooming in makes the layout viewport smaller 
* document - the size of the web page visible or not
  * `document.documentElement.clientWidth` is the width of the viewport including padding but not borders, margins, or vertical scrollbars if present.

* layout viewport - the part of the document rendered in the viewport
* visual viewport - what is visible to the user in the viewport, the layout viewport minus dynamic controls (features that don't scale with the dimensions of a page.) 
  * it is always <= layout viewport

> The visual viewport is the part of the web page that is currently visible in the browser and can change. When the user pinch-zooms the page, pops open a dynamic keyboard, or when a previously hidden address bar becomes visible, the visual viewport shrinks but the layout viewport is unchanged.

*  media queries are relative to the viewport
* viewport length units are a percent of the viewport
  * `width: 50vw;` is 50% of the viewport
  
* `element.getBoundingClientRect();` - returns the size of an element and its position relative to the viewport. The smallest rectangle which contains the entire element [Mozilla's API web docs](https://developer.mozilla.org/en-US/docs/Web/API/Element/getBoundingClientRect)
  * Properties other than width and height are relative to the top-left of the viewport.
  * The amount of scrolling that has been done of the viewport area (or any other scrollable element) is taken into account when computing the bounding rectangle.
  * if `top` >= `window.innerHeight` the element is below the viewport
  * if `bottom` <= `window.innerHeight` the element is above the viewport
  
  The requirement:
  
  * The ISI (Important Safety Information) must always be visible on 25% of the screen.
  * The ISI is embedded in the page, but if that is not 25% visible then display a popup that contains the ISI text and is 25% of the screen size
  
  isiBounds.top + percentVisibleSize > viewportHeight || (isiBounds.bottom - percentVisibleSize) <= viewportHeight

Some code excerpts, the original was in a VueJS component, **TODO**  remove VUE stuff and make it work as plain JS 

```html
  <div id="bounds">
      <div id="content">
        <div>some random content</div>
        <div id="isiContent">this content will also be in the popup.</div>
        <div>some other content</div>
      </div>
      <div id="isiPopup" class="isi-pop-up container row" ref="isiPopup" v-bind:style="'height: ' + isiPopupHeight + 'px'" v-if="config.isAnISI && showISIPopup">
        <div class="isi-container">
          <div id="ISI-popup-nav" ref="isiPopupTopnav" class="isi-top-bar" :style="'background-color: ' + (config.isiTitleBackgroundColor || '#000000') + '; color: ' + (config.isiTitleForegroundColor || '#ffffff')">
            <h4 class="pop-up-header"><strong class="bold-text">{{config.isiTitle}}</strong></h4>
            <button type="button" @click="resizeISI()" class="btn btn-expand-toggle" :class="isISIPopupExpanded?'active':''" :style="'border-color: ' + (config.isiTitleForegroundColor || '#ffffff')"><div class="btn-label" :style="'color: ' + (config.isiTitleForegroundColor || '#ffffff')">^</div></button>
          </div>
          <div class="isi-scroll-container">
            <div class="isi-wrapper">
              <div class="richText">
            </div>
          </div>
        </div>
      </div>
    </div>
```

```ecmascript 6
    // register listeners for viewport changes
    function setupViewportChangeListeners() {
      if (window.addEventListener) {
        window.addEventListener('DOMContentLoaded', this.onViewportChangeHandler, false);
        window.addEventListener('load', this.onViewportChangeHandler, false);
        window.addEventListener('scroll', this.onViewportChangeHandler, false);
        window.addEventListener('resize', this.onViewportChangeHandler, false);
      } else if (window.attachEvent) { // Internet Explorer
        window.attachEvent('onDOMContentLoaded', this.onViewportChangeHandler);
        window.attachEvent('onload', this.onViewportChangeHandler);
        window.attachEvent('onscroll', this.onViewportChangeHandler);
        window.attachEvent('onresize', this.onViewportChangeHandler);
      }
    }

    // unregister listeners for viewport changes
    function removeViewportChangeListeners() {
      if (window.removeEventListener) {
        window.removeEventListener('DOMContentLoaded', this.onViewportChangeHandler, false);
        window.removeEventListener('load', this.onViewportChangeHandler, false);
        window.removeEventListener('scroll', this.onViewportChangeHandler, false);
        window.removeEventListener('resize', this.onViewportChangeHandler, false);
      } else if (window.detachEvent) { // Internet Explorer
        window.detachEvent('onDOMContentLoaded', this.onViewportChangeHandler);
        window.detachEvent('onload', this.onViewportChangeHandler);
        window.detachEvent('onscroll', this.onViewportChangeHandler);
        window.detachEvent('onresize', this.onViewportChangeHandler);
      }
    }

    function onViewportChangeHandler(e) {
      this.adjustISI()
    }

    function resizeISI() {
      this.isISIPopupExpanded = !this.isISIPopupExpanded;
      this.adjustISI();
    };

    function adjustISI() {
      if (this.config.isAnISI && this.$refs.bounds) {
        const inlineISIBounds = this.$refs.bounds.getBoundingClientRect() // bounds of the isi richtext component in the page
        const isiPopupTopnavHeight = (this.$refs.isiPopupTopnav ? this.$refs.isiPopupTopnav.getBoundingClientRect().height : 0);
        const viewportHeight = (window.innerHeight || document.documentElement.clientHeight);
        const percentVisibleSize = (viewportHeight * (this.config.percentVisible || 25) / 100);
        this.showISIPopup = inlineISIBounds.top + percentVisibleSize > viewportHeight || (inlineISIBounds.bottom <= percentVisibleSize);
        if (this.isISIPopupExpanded) {
          this.isiPopupHeight = (viewportHeight - 100);
        } else {
          // add the height of the popup topnav bar
          this.isiPopupHeight = percentVisibleSize + isiPopupTopnavHeight;
        }
      }
    }
```