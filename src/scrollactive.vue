<template>
  <nav class="scrollactive-nav" ref="scrollactive-nav-wrapper">
    <slot></slot>
  </nav>
</template>
<script>
import bezierEasing from 'bezier-easing';

export default {
  props: {
    /**
     * Class that will be applied in the menu item.
     *
     * @default  'is-active'
     * @type {String}
     */
    activeClass: {
      type: String,
      default: 'is-active',
    },

    /**
     * Amount of space between top of screen and the section to
     * highlight. (Usually your fixed header's height)
     *
     * @default 20
     * @type {Number}
     */
    offset: {
      type: Number,
      default: 20,
    },

    /**
     * Enables/disables the scrolling when clicking in a menu item.
     * Disable if you'd like to handle the scrolling by your own.
     *
     * @default true
     * @type {Boolean}
     */
    clickToScroll: {
      type: Boolean,
      default: true,
    },

    /**
     * The duration of the scroll animation when clicking to scroll
     * is activated.
     *
     * @default 600
     * @type {Number}
     */
    duration: {
      type: Number,
      default: 600,
    },

    /**
     * Defines if the plugin should track the section change when
     * clicking an item to scroll to its section. If set to true,
     * it will always keep track and change the active class to the
     * current section while scrolling, if false, the active class
     * will be immediately applied to the clicked menu item, ignoring
     * the passed sections until the scrolling is over.
     *
     * @default false
     * @type {Boolean}
     */
    alwaysTrack: {
      type: Boolean,
      default: false,
    },

    /**
     * Your custom easing value for the click to scroll functionality.
     * It must be a string with 4 values separated by commas in a
     * cubic bezier format.
     *
     * @default '.5,0,.35,1'
     * @type {String}
     */
    bezierEasingValue: {
      type: String,
      default: '.5,0,.35,1',
    },

    /**
     * Decides if the URL should be modified with the section id when
     * clicking a scrollactive item.
     *
     * @default true
     * @type {Boolean}
     */
    modifyUrl: {
      type: Boolean,
      default: true,
    },

    /**
     * If true the active class will only be applied when a section matches exactly one of the
     * scrollactive items, meaning it will be highlighted when scrolling exactly inside the section.
     * If false (default) it will always highlight the last item which was matched in a section,
     * even if it is already outside that section (and not inside another that's being tracked).
     *
     * @default false
     * @type {Boolean}
     */
    exact: {
      type: Boolean,
      default: false,
    },
  },

  data() {
    return {
      observer: null,
      items: [],
      currentItem: null,
      lastActiveItem: null,
      scrollAnimationFrame: null,
      bezierEasing,
    };
  },

  computed: {
    /**
     * Computes the bezier easing string value into an array.
     *
     * @return {Array.<string>}
     */
    cubicBezierArray() {
      return this.bezierEasingValue.split(',');
    },
  },

  mounted() {
    const MutationObserver = window.MutationObserver || window.WebKitMutationObserver;

    if (!this.observer) {
      // Watch for DOM changes in the scrollactive element wrapper
      this.observer = new MutationObserver(this.initScrollactiveItems);
      this.observer.observe(this.$refs['scrollactive-nav-wrapper'], {
        childList: true,
        subtree: true,
      });
    }

    this.initScrollactiveItems();
    this.removeActiveClass();
    this.currentItem = this.getItemInsideWindow();

    if (this.currentItem) this.currentItem.classList.add(this.activeClass);

    window.addEventListener('scroll', this.onScroll);
  },

  updated() {
    this.initScrollactiveItems();
  },

  beforeDestroy() {
    window.removeEventListener('scroll', this.onScroll);
    window.cancelAnimationFrame(this.scrollAnimationFrame);
  },

  methods: {
    /**
     * Will be called when scrolling event is triggered to handle the addition of the active class
     * in the current section item and fire the change event.
     *
     * @param {Object} event Scroll event.
     */
    onScroll(event) {
      this.currentItem = this.getItemInsideWindow();

      if (this.currentItem !== this.lastActiveItem) {
        this.removeActiveClass();
        this.$emit('itemchanged', event, this.currentItem, this.lastActiveItem);
        this.lastActiveItem = this.currentItem;
      }

      // Current item might be null if not inside any section
      if (this.currentItem) this.currentItem.classList.add(this.activeClass);
    },

    /**
     * Gets the scrollactive item that corresponds to the current section inside the window
     *
     * @return {Element} Scrollactive item element.
     */
    getItemInsideWindow() {
      let currentItem;

      [].forEach.call(this.items, (item) => {
        const target = document.getElementById(item.hash.substr(1));
        if (target) {
          const isScreenPastSection = window.pageYOffset >= this.getOffsetTop(target) - this.offset;
          const isScreenBeforeSectionEnd = window.pageYOffset <
            (this.getOffsetTop(target) - this.offset) + target.offsetHeight;

          if (this.exact && isScreenPastSection && isScreenBeforeSectionEnd) currentItem = item;
          if (!this.exact && isScreenPastSection) currentItem = item;
        }

      });

      return currentItem;
    },

    /**
     * Sets the list of menu items, adding or removing the click listener depending on the
     * clickToScroll prop.
     */
    initScrollactiveItems() {
      this.items = this.$el.querySelectorAll('.scrollactive-item');

      if (this.clickToScroll) {
        [].forEach.call(this.items, (item) => {
          item.addEventListener('click', this.handleClick);
        });
        return;
      }

      [].forEach.call(this.items, (item) => {
        item.removeEventListener('click', this.handleClick);
      });
    },

    /**
     * Keep the old setScrollactiveItems method in order to avoid breaking existing projects that
     * used the previous version and upgraded to this one.
     *
     * @deprecated
     */
    setScrollactiveItems() {
      this.initScrollactiveItems();
    },

    /**
     * Handles the scrolling when clicking a menu item.
     *
     * @param {Object} event The click event.
     */
    handleClick(event) {
      event.preventDefault();

      const { hash } = event.currentTarget;
      const target = document.getElementById(hash.substr(1));

      if (!target) {
        console.warn(`[vue-scrollactive] Element '${hash}' was not found. Make sure it is set in the DOM.`);
        return;
      }

      /**
       *  Temporarily removes the scroll listener and the request animation frame so the active
       *  class will only be applied to the clicked element, and not all elements while the window
       *  is scrolling.
       */
      if (!this.alwaysTrack) {
        window.removeEventListener('scroll', this.onScroll);
        window.cancelAnimationFrame(this.scrollAnimationFrame);

        this.removeActiveClass();
        event.currentTarget.classList.add(this.activeClass);
      }

      this.scrollTo(target).then(() => {
        if (this.modifyUrl) {
          // Update the location hash after we've finished animating
          if (window.history.pushState) {
            window.history.pushState(null, null, hash);
          } else {
            window.location.hash = hash;
          }
        }
      });
    },

    /**
     * Scrolls the page to the given target element.
     *
     * @param  {Element} target DOM Element to scroll to.
     * @return {Promise} Returns a promise that will resolve when the animation is done.
     */
    scrollTo(target) {
      return new Promise((resolve) => {
        const targetDistanceFromTop = this.getOffsetTop(target);
        const startingY = window.pageYOffset;
        const difference = targetDistanceFromTop - startingY;
        const easing = this.bezierEasing(...this.cubicBezierArray);
        let start = null;

        const step = (timestamp) => {
          if (!start) start = timestamp;

          let progress = timestamp - start;
          let progressPercentage = progress / this.duration;

          if (progress >= this.duration) progress = this.duration;
          if (progressPercentage >= 1) progressPercentage = 1;

          const perTick = startingY + (easing(progressPercentage) * (difference - this.offset));

          window.scrollTo(0, perTick);

          if (progress < this.duration) {
            this.scrollAnimationFrame = window.requestAnimationFrame(step);
          } else {
            window.addEventListener('scroll', this.onScroll);
            resolve();
          }
        };

        window.requestAnimationFrame(step);
      });
    },

    /**
     * Gets the top offset position of an element in the document.
     *
     * @param  {Element} element
     * @return {Number}
     */
    getOffsetTop(element) {
      let yPosition = 0;
      let nextElement = element;

      while (nextElement) {
        yPosition += (nextElement.offsetTop);
        nextElement = nextElement.offsetParent;
      }

      return yPosition;
    },

    /**
     * Removes the active class from all scrollactive items.
     */
    removeActiveClass() {
      [].forEach.call(this.items, (item) => {
        item.classList.remove(this.activeClass);
      });
    },
  },
};

</script>
