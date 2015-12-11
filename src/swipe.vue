<style>
  .swipe {
    overflow: hidden;
    position: relative;
  }

  .swipe-items-wrap {
    position: relative;
    overflow: hidden;
    height: 100%;
  }

  .swipe-items-wrap > div {
    position: absolute;
    transform: translateX(-100%);
    width: 100%;
    height: 100%;
    display: none;
  }

  .swipe-items-wrap > div.active {
    display: block;
    transform: none;
  }

  .swipe-indicators {
    position: absolute;
    bottom: 10px;
    left: 50%;
    transform: translateX(-50%);
  }

  .swipe-indicator {
    width: 8px;
    height: 8px;
    display: inline-block;
    border-radius: 100%;
    background: #000;
    opacity: 0.2;
    margin: 0 3px;
  }

  .swipe-indicator.active {
    background: #fff;
  }
</style>

<template>
  <div class="swipe">
    <div class="swipe-items-wrap" v-el:wrap>
      <slot></slot>
    </div>
    <div class="swipe-indicators">
      <div class="swipe-indicator" v-for="page in pages" :class="{ active: $index === index }"></div>
    </div>
  </div>
</template>

<script type="text/ecmascript-6">
  import { once, addClass, removeClass } from 'wind-dom';

  var animating = false;

  var translate = function(element, offset, speed, callback) {
    element.style.webkitTransform = `translate3d(${offset}px, 0, 0)`;

    if (speed) {
      animating = true;
      element.style.webkitTransition = '-webkit-transform ' + speed + 'ms ease-in-out';

      once(element, 'transitionend', function() {
        animating = false;
        element.style.webkitTransition = '';
        element.style.webkitTransform = '';
        if (callback) {
          callback.apply(this, arguments);
        }
      });
    } else {
      element.style.webkitTransition = '';
    }
  };

  export default {
    created() {
      this.dragState = {};
    },

    data() {
      return {
        ready: false,
        dragging: false,
        index: 0,
        pages: [],
        timer: null,
        reInitTimer: null
      }
    },

    props: {
      speed: {
        type: Number,
        default: 300
      },

      auto: {
        type: Number,
        default: 3000
      },

      continuous: {
        type: Boolean,
        default: true
      }
    },

    events: {
      swipeItemCreated() {
        if (!this.ready) return;

        clearTimeout(this.reInitTimer);
        this.reInitTimer = setTimeout(() => {
          this.reInitPages();
        }, 100);
      },

      swipeItemDestroyed(){
        if (!this.ready) return;

        clearTimeout(this.reInitTimer);
        this.reInitTimer = setTimeout(() => {
          this.reInitPages();
        }, 100);
      }
    },

    methods: {
      reInitPages() {
        var children = this.$children;
        var pages = [];

        children.forEach(function(child, index) {
          pages.push(child.$el);

          if (index === 0) {
            addClass(child.$el, 'active');
          }
        });

        this.pages = pages;
      },

      doAnimate(towards, options) {
        var prevPage, nextPage, currentPage, pageWidth;
        var speed = this.speed || 300;
        var index = this.index;
        var pages = this.pages;
        var pageCount = pages.length;

        if (!options) {
          pageWidth = this.$el.clientWidth;
          currentPage = pages[index];
          prevPage = pages[index - 1];
          nextPage = pages[index + 1];
          if (this.continuous && pages.length > 1) {
            if (!prevPage) {
              prevPage = pages[pages.length - 1];
            }
            if (!nextPage) {
              nextPage = pages[0];
            }
          }
          if (prevPage) {
            prevPage.style.display = 'block';
            translate(nextPage, -pageWidth);
          }
          if (nextPage) {
            nextPage.style.display = 'block';
            translate(nextPage, pageWidth);
          }
        } else {
          prevPage = options.prevPage;
          currentPage = options.currentPage;
          nextPage = options.nextPage;
          pageWidth = options.pageWidth;
        }

        var newIndex;

        var oldPage = this.$children[index].$el;

        if (towards === 'prev') {
          if (index > 0) {
            newIndex = index - 1;
          }
          if (this.continuous && index === 0) {
            newIndex = pageCount - 1;
          }
        } else if (towards === 'next') {
          if (index < pageCount - 1) {
            newIndex = index + 1;
          }
          if (this.continuous && index === pageCount - 1) {
            newIndex = 0;
          }
        }

        var callback = () => {
          if (newIndex !== undefined) {
            var newPage = this.$children[newIndex].$el;
            removeClass(oldPage, 'active');
            addClass(newPage, 'active');

            this.index = newIndex;
          }

          if (prevPage) {
            prevPage.style.display = '';
          }

          if (nextPage) {
            nextPage.style.display = '';
          }
        };

        setTimeout(() => {
          if (towards === 'next') {
            translate(currentPage, -pageWidth, speed, callback);
            if (nextPage) {
              translate(nextPage, 0, speed);
            }
          } else if (towards === 'prev') {
            translate(currentPage, pageWidth, speed, callback);
            if (prevPage) {
              translate(prevPage, 0, speed);
            }
          } else {
            translate(currentPage, 0, speed, callback);
            if (prevPage) {
              translate(prevPage, pageWidth * -1, speed);
            }
            if (nextPage) {
              translate(nextPage, pageWidth, speed);
            }
          }
        }, 10);
      },

      next() {
        this.doAnimate('next');
      },

      prev() {
        this.doAnimate('prev');
      },

      doOnTouchStart: function(event) {
        var swipe = this;
        var element = swipe.$el;
        var dragState = swipe.dragState;
        var touch = event.touches[0];

        dragState.startTime = new Date();
        dragState.startLeft = touch.pageX;
        dragState.startTop = touch.pageY;

        dragState.pageWidth = element.offsetWidth;
        dragState.pageHeight = element.offsetHeight;

        var prevPage = this.$children[this.index - 1];
        var dragPage = this.$children[this.index];
        var nextPage = this.$children[this.index + 1];

        if (this.continuous && this.pages.length > 1) {
          if (!prevPage) {
            prevPage = this.$children[this.$children.length - 1];
          }
          if (!nextPage) {
            nextPage = this.$children[0];
          }
        }

        dragState.prevPage = prevPage ? prevPage.$el : null;
        dragState.dragPage = dragPage ? dragPage.$el : null;
        dragState.nextPage = nextPage ? nextPage.$el : null;

        if (dragState.prevPage) {
          dragState.prevPage.style.display = 'block';
        }

        if (dragState.nextPage) {
          dragState.nextPage.style.display = 'block';
        }
      },

      doOnTouchMove: function(event) {
        var swipe = this;
        var dragState = swipe.dragState;
        var touch = event.touches[0];

        dragState.currentLeft = touch.pageX;
        dragState.currentTop = touch.pageY;

        var offsetTop = dragState.currentTop - dragState.startTop;
        var offsetLeft = dragState.currentLeft - dragState.startLeft;

        if (dragState.prevPage) {
          translate(dragState.prevPage, offsetLeft - dragState.pageWidth);
        }
        translate(dragState.dragPage, offsetLeft);
        if (dragState.nextPage) {
          translate(dragState.nextPage, offsetLeft + dragState.pageWidth);
        }
      },

      doOnTouchEnd: function() {
        var swipe = this;
        var dragState = swipe.dragState;

        var dragDuration = new Date() - dragState.startTime;
        var towards = null;

        var offsetLeft = dragState.currentLeft - dragState.startLeft;
        var pageWidth = dragState.pageWidth;
        var index = this.index;
        var pageCount = this.pages.length;

        if (dragDuration < 300 || Math.abs(offsetLeft) > pageWidth / 2) {
          towards = offsetLeft < 0 ? 'next' : 'prev';
        }

        if (!this.continuous) {
          if ((index === 0 && towards === 'prev') || (index === pageCount - 1 && towards === 'next')) {
            towards = null;
          }
        }

        this.doAnimate(towards, {
          pageWidth: dragState.pageWidth,
          prevPage: dragState.prevPage,
          currentPage: dragState.dragPage,
          nextPage: dragState.nextPage
        });

        this.dragState = {};
      }
    },

    destroyed() {
      if (this.timer) {
        clearInterval(this.timer);
        this.timer = null;
      }
    },

    ready() {
      this.ready = true;

      if (this.auto > 0) {
        this.timer = setInterval(() => {
          if (!this.dragging) {
            this.next();
          }
        }, this.auto);
      }

      var element = this.$el;

      element.addEventListener('touchstart', (event) => {
        if (animating) return;
        this.dragging = true;
        this.doOnTouchStart(event);
      });

      element.addEventListener('touchmove', (event) => {
        if (!this.dragging) return;
        this.doOnTouchMove(event);
      });

      element.addEventListener('touchend', (event) => {
        if (!this.dragging) return;
        this.doOnTouchEnd(event);
        this.dragging = false;
      });

      this.reInitPages();
    }
  };
</script>