<template>
  <div>
    <div class="timeline-title">
          <div class="timeline-title__time">{{ timelineTitle.timeFrom }}{{ timelineTitle.timeTo && ` ~ ${timelineTitle.timeTo}` }}</div>
          <div class="timeline-title__position">{{ timelineTitle.places.join(",") }}</div>
        </div>
        <div ref="container" :style="{height: `${this.height}px`}" grid-list-xs fluid class="p-photos p-photo-mosaic p-photo-timeline">

<div v-if="totalCount === 0">
  <v-alert
      :value="true"
      color="secondary-dark"
      :icon="isSharedView ? 'image_not_supported' : 'lightbulb_outline'"
      class="no-results ma-2 opacity-70"
      outline
  >
    <h3>
      <translate>No pictures found</translate>
    </h3>

  </v-alert>
</div>
<div class="search-results photo-results mosaic-view timeline-view" :class="{'select-results': selectMode}">

  <div
      v-for="(photo, index) in photos"
      v-if="photo && isPhotoVisible(index)"
      ref="items"
      :key="index"
      :data-index="index"
      :style="getPhotoStyle(index)"
  >
   <!--
     The following div is the layout + size container. It makes the browser not
     re-layout all elements in the list when the children of one of them changes
    -->
    <div class="image-container">
      <!-- <div v-if="index < firstVisibleElementIndex || index > lastVisibileElementIndex"
           :data-uid="photo.UID"
           class="card darken-1 result image"
      ></div> -->
      <div  
            :key="photo.Hash"
            tile
            :data-id="photo.ID"
            :data-uid="photo.UID"
            :style="`background-image: url(${photo.thumbnailUrl('tile_224')});`"
            :class="photo.classes().join(' ') + ' card darken-1 result clickable image'"
            :alt="photo.Title"
            :title="photo.Title"
            @contextmenu.stop="onContextMenu($event, index)"
            @touchstart.passive="input.touchStart($event, index)"
            @touchend.stop.prevent="onClick($event, index)"
            @mousedown.stop.prevent="input.mouseDown($event, index)"
            @click.stop.prevent="onClick($event, index)"
            @mouseover="playLive(photo)"
            @mouseleave="pauseLive(photo)">
        <div v-if="photo.Type === 'live' || photo.Type === 'animated'" class="live-player">
          <video :id="'live-player-' + photo.ID" :key="photo.ID" width="224" height="224" preload="none"
                 loop muted playsinline>
            <source :src="photo.videoUrl()">
          </video>
        </div>

        <button v-if="photo.Type !== 'image' || photo.isStack()"
              class="input-open"
              @touchstart.stop.prevent="input.touchStart($event, index)"
              @touchend.stop.prevent="onOpen($event, index, !isSharedView, photo.Type === 'live')"
              @touchmove.stop.prevent
              @click.stop.prevent="onOpen($event, index, !isSharedView, photo.Type === 'live')">
          <i v-if="photo.Type === 'raw'" class="action-raw" :title="$gettext('RAW')">raw_on</i>
          <i v-if="photo.Type === 'live'" class="action-live" :title="$gettext('Live')"><icon-live-photo/></i>
          <i v-if="photo.Type === 'video'" class="action-play" :title="$gettext('Video')">play_arrow</i>
          <i v-if="photo.Type === 'animated'" class="action-animated" :title="$gettext('Animated')">gif</i>
          <i v-if="photo.Type === 'vector'" class="action-vector" :title="$gettext('Vector')">font_download</i>
          <i v-if="photo.Type === 'image'" class="action-stack" :title="$gettext('Stack')">burst_mode</i>
        </button>

        <button v-if="photo.Type === 'image' && selectMode"
              class="input-view"
              :title="$gettext('View')"
              @touchstart.stop.prevent="input.touchStart($event, index)"
              @touchend.stop.prevent="onOpen($event, index)"
              @touchmove.stop.prevent
              @click.stop.prevent="onOpen($event, index)">
          <i color="white" class="action-fullscreen">zoom_in</i>
        </button>

        <button v-if="!isSharedView && hidePrivate && photo.Private" class="input-private">
          <i color="white" class="select-on">lock</i>
        </button>

        <!--
          We'd usually use v-if here to only render the button if needed.
          Because the button is supposed to be visible when the result is
          being hovered over, implementing the v-if would require the use of
          a <v-hover> element around the result.

          Because rendering the plain HTML-Button is faster than rendering
          the v-hover component we instead hide the button by default and
          use css to show it when it is being hovered.
        -->
        <button
              class="input-select"
              @mousedown.stop.prevent="input.mouseDown($event, index)"
              @touchstart.stop.prevent="input.touchStart($event, index)"
              @touchend.stop.prevent="onSelect($event, index)"
              @touchmove.stop.prevent
              @click.stop.prevent="onSelect($event, index)">
          <i color="white" class="select-on">check_circle</i>
          <i color="white" class="select-off">radio_button_off</i>
        </button>

        <button v-if="!isSharedView"
            class="input-favorite"
            @touchstart.stop.prevent="input.touchStart($event, index)"
            @touchend.stop.prevent="toggleLike($event, index)"
            @touchmove.stop.prevent
            @click.stop.prevent="toggleLike($event, index)"
        >
          <i v-if="photo.Favorite">favorite</i>
          <i v-else>favorite_border</i>
        </button>
      </div>
    </div>
  </div>
</div>
<p-photo-time-wheel
  :ids="ids"
  :photos="photos"
  :start-index="photoVisibleArea.startPhotoCount"
  :end-index="photoVisibleArea.endPhotoCount"
  :ensurePhotoLoaded="ensurePhotoLoaded"
  :onNavigateByPercentage="onNavigateByPercentage"
></p-photo-time-wheel>
</div>
  </div>

  </template>
  <script>
  import {Input, InputInvalid, ClickShort, ClickLong} from "common/input";
  import IconLivePhoto from "component/icon/live-photo.vue";
  import PPhotoTimeWheel from "./photo-time-wheel.vue";

  export default {
    name: 'PPhotoTimeline',
    components: {
      IconLivePhoto,
      PPhotoTimeWheel
    },
    props: {
      photos: {
        type: Array,
        default: () => [],
      },
      ids: {
        type: Array,
        default: () => [],
      },
      openPhoto: {
        type: Function,
        default:() => {},
      },
      editPhoto: {
        type: Function,
        default: () => {},
      },
      album: {
        type: Object,
        default: () => {},
      },
      filter: {
        type: Object,
        default: () => {},
      },
      context: {
        type: String,
        default: "",
      },
      totalCount: {
        type: Number,
        default: undefined,
      },
      photosPerLine: {
        type: Number,
        default: 10
      },
      ensurePhotoLoaded: {
        type: Function,
        default: (start, end) => {}
      },
      selectMode: Boolean,
      isSharedView: {
        type: Boolean,
        default: false,
      },
    },
    data() {
      return {
        hidePrivate: this.$config.settings().features.private,
        input: new Input(),
        firstVisibleElementIndex: 0,
        lastVisibileElementIndex: 0,
        width: 0,
        scrollTop: 0,
        visibleElementIndices: new Set(),
      };
    },
    watch: {
      // photos: {
      //   handler() {
      //     this.$nextTick(() => {
      //       this.observeItems();
      //     });
      //   },
      //   immediate: true,
      // }
    },
    computed: {
      lineCount() {
          return Math.ceil(this.totalCount / this.photosPerLine);
        },
      lineHeight() {
          return Math.floor(this.width / this.photosPerLine);
        },
      height() {
          const result = this.lineCount * this.lineHeight;
          return result;
        },
      photoVisibleArea (){
          let scrollTop = Math.max(0, this.scrollTop);
          let visibleLines = Math.ceil(window.innerHeight / this.lineHeight);
          let startLineCount = Math.floor(scrollTop / this.lineHeight);
          let endLineCount = startLineCount + visibleLines;

          let bufferLines = 1;
          startLineCount = Math.max(0, startLineCount - bufferLines);
          endLineCount = endLineCount + bufferLines;
          let startPhotoCount = startLineCount * this.photosPerLine;
          let endPhotoCount = endLineCount * this.photosPerLine - 1;
          return {
            startLineCount,
            endLineCount,
            startPhotoCount,
            endPhotoCount
          }
        },
      headingHeight() {
        const titleHeight = document.querySelector(".timeline-title").scrollHeight;
        return titleHeight;
      },

      timelineTitle(){
        const allVisiblePhotos = this.photos.filter((p, i) => this.isPhotoVisible(i));
        const result = {
          places: [],
          timeFrom: 0,
          timeTo: 0
        };
        allVisiblePhotos.reduce((acc, cur) => {
          if (!cur){
            return acc;
          }
          if (!acc.places.includes(cur.locationInfo())){
            acc.places.push(cur.locationInfo());
          }

          return acc;
        }, result);

        result.timeFrom = allVisiblePhotos[0]?.shortDateString();
        result.timeTo = allVisiblePhotos[allVisiblePhotos.length - 1]?.shortDateString();
        if (result.timeFrom === result.timeTo){
          result.timeTo = undefined;
        }
        return result;
      }
    },
    beforeCreate() {
      // this.intersectionObserver = new IntersectionObserver((entries) => {
      //   this.visibilitiesChanged(entries);
      // }, {
      //   rootMargin: "50% 0px",
      // });
    },

    mounted(){
      this.bindResizeEvent();
      const rect = this.$refs.container.getBoundingClientRect();
      this.width = rect.width;
      window.addEventListener("scroll", this.onScroll);
      this.updateDisplayPhotos();
    },
    created() {

    },
    beforeDestroy() {
      window.removeEventListener("scroll", this.onScroll)
      // this.intersectionObserver.disconnect();
      this.unbindResizeEvent();
    },
    methods: {
      async updateDisplayPhotos (){
        const photoVisibleArea = this.photoVisibleArea;
        await this.ensurePhotoLoaded(photoVisibleArea.startPhotoCount, photoVisibleArea.endPhotoCount);
      },

      isPhotoVisible(i){
        return (i >= this.photoVisibleArea.startPhotoCount && i <= this.photoVisibleArea.endPhotoCount);
      },

      getPhotoStyle(i){
        const result = {
          position: "absolute",
          width: `${this.lineHeight}px`,
          height: `${this.lineHeight}px`,
          left: `${(i % this.photosPerLine) * this.lineHeight}px`,
          top: `${(parseInt(i / this.photosPerLine)) * this.lineHeight}px`
        };

        return result;
      },
      async onScrollDebounced(e){
        var scrollTop = window.scrollY;
        var headingHeight = this.headingHeight;
        
        this.scrollTop = scrollTop - headingHeight;
        console.log(this);
        await this.updateDisplayPhotos();
      },
      onScroll (e) {
        clearTimeout(this._onscrollTimeout);
        this._onscrollTimeout = setTimeout(() => {this.onScrollDebounced(e)}, 10)
      },
      onContainerResize(width, height){
        this.width = width;
      },
      bindResizeEvent(){
        if (!this.resizeObserver){
          var resizeObserver = new ResizeObserver((enteries) => {
            let entry = enteries[0];
            const width = entry.contentRect.width;
            const height = entry.contentRect.height;
            this.onContainerResize(width, height);
          });
          this.resizeObserver = resizeObserver;
        }
        this.resizeObserver.observe(this.$refs.container);
      },

      unbindResizeEvent(){
        if (!this.resizeObserver){
          return;
        }
        this.resizeObserver.unobserve(this.$refs.container);
      },
      onNavigateByPercentage(data){
          document.documentElement.scrollTop = (document.documentElement.offsetHeight) * data.scrollTo; 
      },

      // observeItems() {
      //   if (this.$refs.items === undefined) {
      //     return;
      //   }
  
      //   /**
      //    * observing only every 5th item reduces the amount of time
      //    * spent computing intersection by 80%. me might render up to
      //    * 8 items more than required, but the time saved computing
      //    * intersections is far greater than the time lost rendering
      //    * a couple more items
      //    */
      //   for (let i = 0; i < this.$refs.items.length; i += 5) {
      //     this.intersectionObserver.observe(this.$refs.items[i]);
      //   }
      // },
      // elementIndexFromIntersectionObserverEntry(entry) {
      //   return parseInt(entry.target.getAttribute('data-index'));
      // },
      // visibilitiesChanged(entries) {
      //   const [smallestIndex, largestIndex] = virtualizationTools.updateVisibleElementIndices(
      //     this.visibleElementIndices,
      //     entries,
      //     this.elementIndexFromIntersectionObserverEntry,
      //   );
  
      //   // we observe only every 5th item, so we increase the rendered
      //   // range here by 4 items in every directio just to be safe
      //   this.firstVisibleElementIndex = smallestIndex - 4;
      //   this.lastVisibileElementIndex = largestIndex + 4;
      // },
      livePlayer(photo) {
        return document.querySelector("#live-player-" + photo.ID);
      },
      playLive(photo) {
        const player = this.livePlayer(photo);
        try { if (player) player.play(); }
        catch (e) {
          // Ignore.
        }
      },
      pauseLive(photo) {
        const player = this.livePlayer(photo);
        try { if (player) player.pause(); }
        catch (e) {
          // Ignore.
        }
      },
      toggleLike(ev, index) {
        const inputType = this.input.eval(ev, index);
  
        if (inputType !== ClickShort) {
          return;
        }
  
        const photo = this.photos[index];
  
        if (!photo) {
          return;
        }
  
        photo.toggleLike();
      },
      onSelect(ev, index) {
        const inputType = this.input.eval(ev, index);
  
        if (inputType !== ClickShort) {
          return;
        }
  
        if (ev.shiftKey) {
          this.selectRange(index);
        } else {
          this.toggle(this.photos[index]);
        }
      },
      toggle(photo) {
        this.$clipboard.toggle(photo);
        this.$forceUpdate();
      },
      onOpen(ev, index, showMerged, preferVideo) {
        const inputType = this.input.eval(ev, index);
  
        if (inputType !== ClickShort) {
          return;
        }
  
        this.openPhoto(index, showMerged, preferVideo);
      },
      onClick(ev, index) {

        const inputType = this.input.eval(ev, index);
        const longClick = inputType === ClickLong;
  
        if (inputType === InputInvalid) {
          return;
        }
  
        if (longClick || this.selectMode) {
          if (longClick || ev.shiftKey) {
            this.selectRange(index);
          } else {
            this.toggle(this.photos[index]);
          }
        } else {
          this.openPhoto(index);
        }
      },
      onContextMenu(ev, index) {
        if (this.$isMobile) {
          ev.preventDefault();
          ev.stopPropagation();
          this.selectRange(index);
        }
      },
      selectRange(index) {
        this.$clipboard.addRange(index, this.photos);
        this.$forceUpdate();
      }
    },
  };
  </script>