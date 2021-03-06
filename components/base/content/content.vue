<template>
    <article content class="ion-content">
        <section ref="fixedElement" class="fixed-content" :style="fixedElementStyle">
            <!--Fixed-->
            <slot name="fixed"></slot>
            <!--Fixed Top-->
            <slot name="fixedTop"></slot>
            <!--Fixed Bottom-->
            <slot name="fixedBottom"></slot>
        </section>
        <section ref="scrollElement" class="scroll-content" :style="scrollElementStyle">
            <!--默认是能滚动的内容-->
            <!--原生滚动-->
            <slot></slot>
        </section>
        <slot name="refresher"></slot>
    </article>
</template>
<style lang="less">
    @import "content";
</style>
<script type="text/javascript">
  /**
   * @typedef {Object} ContentDimension   - Content组件的维度尺寸信息
   * @property {number} contentHeight     - content offsetHeight,           content自身高度
   * @property {number} contentTop        - content offsetTop,               content到窗体顶部的距离
   * @property {number} contentBottom     - content offsetTop+offsetHeight,  content底部到窗体顶部的的距离
   * @property {number} contentWidth      - content offsetWidth
   * @property {number} contentLeft       - content contentLeft
   * @property {number} contentRight      - content offsetLeft + offsetWidth
   * @property {number} scrollHeight      - scroll scrollHeight
   * @property {number} scrollTop         - scroll scrollTop
   * @property {number} scrollBottom      - scroll scrollTop + scrollHeight
   * @property {number} scrollWidth       - scroll scrollWidth
   * @property {number} scrollLeft        - scroll scrollLeft
   * @property {number} scrollRight       - scroll scrollLeft + scrollWidth
   * */

  /**
   * @typedef {Object} ScrollEvent            - 滚动事件返回的滚动对象
   * @property {number} timeStamp             - 滚动事件
   * @property {number} scrollTop             -
   * @property {number} scrollLeft            -
   * @property {number} scrollHeight          -
   * @property {number} scrollWidth           -
   * @property {number} contentHeight         -
   * @property {number} contentWidth          -
   * @property {number} contentTop            -
   * @property {number} contentBottom         -
   * @property {number} startY                -
   * @property {number} startX                -
   * @property {number} deltaY                -
   * @property {number} deltaX                -
   * @property {number} velocityY             -
   * @property {number} velocityX             -
   * @property {number} directionY            -
   * @property {number} directionX            -
   * @property {HTMLElement} contentElement   -
   * @property {HTMLElement} fixedElement     -
   * @property {HTMLElement} scrollElement    -
   * @property {HTMLElement} scrollElement    -
   * @property {HTMLElement} headerElement    -
   * @property {HTMLElement} footerElement    -
   * */

  /**
   * @component Base/Content
   * @description
   *
   * ## 基础组件 / Content组件
   *
   * Vimo框架的页面基础布局分为Header/Content/Footer三个部分, 也就是"上中下三明治"结构布局, Content组件则是中间业务内容的位置.
   *
   * Content组件中书写的代码可以是滚动的内容, 也可以是固定在一面不随滚动的内容, 比如说当页的广告/刷新按钮/歌词等. 这个特性的的开启需要特殊命名slot才能激活.
   *
   * 此外需要注意的是, 一个页面(Page组件)中只能有一个Content组件, 这个是Vimo使用的规则!
   *
   * Content组件中也可以加入下拉刷新和上拉加载的功能, 具体请参考示例.
   *
   * ## 不需要引入
   *
   * 是的, 基础组件是安装vimo后自动全局注册的.
   *
   * @demo #/content
   *
   * @slot 空                slot为空则将内容插入到scroll中
   * @slot [fixed]          默认值, 固定到顶部
   * @slot [fixedTop]       固定到顶部
   * @slot [fixedBottom]    固定到底部
   * @slot [refresher]      refresher组件的位置
   *
   * @props {boolean} [fullscreen=false] - 控制Content是否全屏显示, 如果为true, 则Content的上下将延伸到Header和Footer的下面
   * @props {boolean} [recordPosition=false] - 控制Content组件是否记录其浏览位置, 这个可在config中设置, 默认为false, 建议无限滚动页面设置false
   *
   * @fires component:Base/Content#onScrollStart
   * @fires component:Base/Content#onScroll
   * @fires component:Base/Content#onScrollEnd
   *
   *
   * @usage
   * <template>
   *  <Page>
   *    <Header>
   *      <Navbar>
   *        <Title>Demo</Title>
   *      <Navbar>
   *    </Header>
   *    <Content record-position>
   *      <h1>这里是内容</h1>
   *      <p>滚动位置将会被记录</p>
   *    </Content>
   *  </Page>
   * </template>
   *
   * */
  import { transitionEnd, parsePxUnit } from '../../util/util'
  import { ScrollView } from './scroll-view'

  export default {
    name: 'Content',
    props: {
      fullscreen: Boolean,
      recordPosition: {
        type: Boolean,
        default () { return this.$config && this.$config.getBoolean('recordPosition', false) || false }
      }
    },
    data () {
      return {
        isFullscreen: this.fullscreen,

        fixedElementStyle: {},  // 固定内容的位置样式
        scrollElementStyle: {}, // 滑动内容的位置样式

        scrollElement: null,    // scrollConent的DOM句柄
        fixedElement: null,     // fixedElement的DOM句柄

        headerElement: null,    // Header组件的DOM句柄
        footerElement: null,    // footer组件的DOM句柄
        tabsElement: null,      // tabs组件的DOM句柄

        headerBarHeight: 0,
        footerBarHeight: 0,

        _scroll: null,  // 滚动的实例
        _cTop: 0,       // content top
        _cBottom: 0,    // content bottom
        _fTop: 0,       // fixex top
        _fBottom: 0,    // fixex bottom
        _pTop: 0,       // padding top
        _pBottom: 0,    // padding bottom

        _imgs: [],      // 子组件Img的实例列表
        _imgReqBfr: 0,  // 1400
        _imgRndBfr: 0,  // 400
        _imgVelMax: 0
      }
    },
    watch: {
      // 当值改变是，重新设置
      fullscreen () {
        this.isFullscreen = this.fullscreen
        this.recalculateContentDimensions()
      }
    },
    methods: {
      // -------- public --------
      /**
       * @function getContentDimensions
       * @description
       * 返回滚动元素的维度信息
       * @return {ContentDimension}
       * */
      getContentDimensions () {
        console.assert(this._scroll, 'The method getContentDimensions() need _scroll instance, please check!')
        if (!this._scroll) return
        const scrollEle = this.scrollElement
        const parentElement = scrollEle.parentElement
        let contentHeight = parentElement.offsetHeight - this._cTop - this._cBottom
        let contentTop = this._cTop
        let contentBottom = parentElement.offsetHeight - this._cBottom
        let contentWidth = parentElement.offsetWidth
        let contentLeft = parentElement.offsetLeft

        return {
          contentHeight,
          contentTop,
          contentBottom,
          contentWidth,
          contentLeft,

          scrollHeight: this._scroll.getHeight(),
          scrollTop: this._scroll.getTop(),
          scrollWidth: this._scroll.getWidth(),
          scrollLeft: this._scroll.getLeft()
        }
      },
      /**
       * @function resize
       * @description
       * 当动态添加Header/Footer/Tabs或者修改了他的属性时, 重新计算Content组件的尺寸.
       * */
      resize () {
        // 等待DOM更新完毕
        this.$nextTick(() => {
          this.recalculateContentDimensions()
        })
      },
      /**
       * @function scrollTo
       * @description
       * 滚动到指定位置
       * @param {Number} [x=0]            - 滚动到指定位置的x值
       * @param {Number} [y=0]            - 滚动到指定位置的y值
       * @param {Number} [duration=300]   - 滚动动画的时间
       * @param {Function=} done          - 当滚动结束时触发的回调
       * @return {Promise}                - 当回调done未定义的时候, 才返回Promise, 如果定义则返回undefined
       * */
      scrollTo (x, y, duration = 300, done) {
        return this._scroll.scrollTo(x, y, duration, done)
      },
      /**
       * @function scrollToTop
       * @description
       * 滚动到顶部
       * @param {Number} [duration=300] - 滚动动画的时间, 默认是300ms
       * @return {promise} 当滚动动画完毕后返回promise
       */
      scrollToTop (duration = 300) {
        // 页面防止点击
        this.$app && this.$app.setDisableScroll(true, duration)
        return this._scroll.scrollToTop(duration)
      },
      /**
       *
       * @function scrollToBottom
       * @description
       * 滚动到顶部
       * @param {Number} [duration=300] - 滚动动画的时间, 默认是300ms
       * @return {promise} 当滚动动画完毕后返回promise
       */
      scrollToBottom (duration = 300) {
        // 页面防止点击
        this.$app && this.$app.setDisableScroll(true, duration)
        return this._scroll.scrollToBottom(duration)
      },

      /**
       * @function scrollBy
       * @description
       * 滚动到指定位置, 这个和scrollTo类似, 只不过是相对当前位置的滚动
       *
       * ```
       * 当前位置为scrollTop为`100px`, 执行`myScroll.scrollBy(0, -10)`, 则滚动到`110px`位置
       * ```
       *
       * @param {Number} x                - 滚动到指定位置的x值
       * @param {Number} y                - 滚动到指定位置的y值
       * @param {Number} [duration=300]   - 滚动动画的时间
       * @param {Function=} done          - 当滚动结束时触发的回调
       * @return {Promise}                - 当回调done未定义的时候, 才返回Promise, 如果定义则返回undefined
       * */
      scrollBy (x, y, duration = 300, done) {
        this.$app && this.$app.setDisableScroll(true, duration)
        return this._scroll.scrollBy(x, y, duration, done)
      },

      /**
       * @function scrollToElement
       * @description
       * 滚动到指定元素
       * @param {Number} el
       * @param {Number} [duration=300]   - 滚动动画的时间
       * @param {Number} [offsetX=0]
       * @param {Number} [offsetY=0]
       * @param {Function=} done          - 当滚动结束时触发的回调
       * @return {Promise}                - 当回调done未定义的时候, 才返回Promise, 如果定义则返回undefined
       * */
      scrollToElement (el, duration = 300, offsetX = 0, offsetY = 0, done) {
        this.$app && this.$app.setDisableScroll(true, duration)
        return this._scroll.scrollToElement(el, duration, offsetX, offsetY, done)
      },

      // -------- private --------

      /**
       * DOM完毕后进行初始化
       * @private
       * */
      init () {
        if (this.scrollElement) return

        const scroll = this._scroll // 滚动的实例

        /**
         * 找到fixedElement/scrollElement的位置
         * */
        scroll.ev.fixedElement = this.fixedElement = this.$refs.fixedElement
        scroll.ev.scrollElement = this.scrollElement = this.$refs.scrollElement

        /**
         * @event component:Base/Content#onScrollStart
         * @description 滚动开始时触发的事件
         * @property {ScrollEvent} ev - 滚动事件对象
         */
        scroll.scrollStart = (ev) => {
          this.$emit('onScrollStart', ev)
          this.$eventBus && this.$eventBus.$emit('onScrollStart', ev)
        }

        /**
         * @event component:Base/Content#onScroll
         * @description 滚动时触发的事件
         * @property {ScrollEvent} ev - 滚动事件对象
         */
        scroll.scroll = (ev) => {
          this.$emit('onScroll', ev)
          this.$eventBus && this.$eventBus.$emit('onScroll', ev)

          // img更新
          this.imgsUpdate()

          this.recordScrollPosition(ev)
        }

        /**
         * @event component:Base/Content#onScrollEnd
         * @description 滚动结束时触发的事件
         * @property {ScrollEvent} ev - 滚动事件对象
         */
        scroll.scrollEnd = (ev) => {
          this.$emit('onScrollEnd', ev)
          this.$eventBus && this.$eventBus.$emit('onScrollEnd', ev)

          // img更新
          this.imgsUpdate()
        }

        /**
         * 计算并设置当前Content的位置及尺寸
         * */
        this.recalculateContentDimensions()
      },

      /**
       * 重新计算Content组件的尺寸维度
       * 因为这部分受以下因素影响：fullscreen、Header，Footer
       * @private
       * */
      recalculateContentDimensions () {
        const scrollEle = this.scrollElement
        if (!scrollEle) {
          console.assert(false, 'this.scrollElement should be valid')
          return
        }

        const fixedEle = this.fixedElement
        if (!fixedEle) {
          console.assert(false, 'this.fixedElement should be valid')
          return
        }

        // 如果滚动实例_scroll不存在
        // 则直接返回
        if (!this._scroll) return
        let scrollEvent = this._scroll.ev

        let ele = this.$el // HTMLElement

        // 获取Footer/Header信息
        let computedStyle
        let tagName
        let children = this.$parent.$children
        children.forEach((child) => {
          ele = child.$el
          tagName = child.$options._componentTag.toLowerCase()
          if (tagName === 'header') {
            this.headerElement = scrollEvent.headerElement = ele
            computedStyle = window.getComputedStyle(this.headerElement)
            this.headerBarHeight = parsePxUnit(computedStyle.height)
          } else if (tagName === 'footer') {
            this.footerElement = scrollEvent.footerElement = ele
            computedStyle = window.getComputedStyle(this.footerElement)
            this.footerBarHeight = parsePxUnit(computedStyle.height)
          }
        })

        // 获取Content信息
        scrollEvent.contentElement = this.$el
        if (this.isFullscreen) {
          computedStyle = window.getComputedStyle(this.$el)
          this._pTop = parsePxUnit(computedStyle.paddingTop)
          this._pBottom = parsePxUnit(computedStyle.paddingBottom)
        }

        // Content的位置值
        this._cTop = this.headerBarHeight // contentTop
        this._cBottom = this.footerBarHeight

        // Fixed的位置值和Content相等, 但是属性可能不一样
        this._fTop = this._cTop
        this._fBottom = this._cBottom

        // 如果是fullscreen的话,padding还需要计算之前的值, 一般为16px
        if (this.isFullscreen) {
          this._cTop += this._pTop
          this._cBottom += this._pBottom
        }

        // 默认为fullscreen未开启状态, 使用margin属性
        let topProperty = 'marginTop'
        let bottomProperty = 'marginBottom'
        let fixedTop = this._fTop
        let fixedBottom = this._fBottom

        // 如果fullscreen开启, 使用padding属性
        if (this.isFullscreen) {
          console.assert(this._pTop >= 0, '_paddingTop has to be positive')
          console.assert(this._pBottom >= 0, '_paddingBottom has to be positive')

          // 调整Content组件的padding属性, 使得content中的内容在Header和Footer下方滚动,
          topProperty = 'paddingTop'
          bottomProperty = 'paddingBottom'
        }

        // update top margin if value changed
        console.assert(this._cTop >= 0, 'contentTop has to be positive')
        console.assert(fixedTop >= 0, 'fixedTop has to be positive');

        (scrollEle.style)[topProperty] = cssFormat(this._cTop)
        fixedEle.style.marginTop = cssFormat(fixedTop)

        // update bottom margin if value changed
        console.assert(this._cBottom >= 0, 'contentBottom has to be positive')
        console.assert(fixedBottom >= 0, 'fixedBottom has to be positive');

        (scrollEle.style)[bottomProperty] = cssFormat(this._cBottom)
        fixedEle.style.marginBottom = cssFormat(fixedBottom)

        // 初始化_scroll滚动对象
        this._scroll.init(this.scrollElement)

        // 计算Content组件的维度信息, 写入scrollEvent中, 只是初始化的信息
        const contentDimensions = this.getContentDimensions()
        scrollEvent.scrollHeight = contentDimensions.scrollHeight
        scrollEvent.scrollWidth = contentDimensions.scrollWidth
        scrollEvent.contentHeight = contentDimensions.contentHeight
        scrollEvent.contentWidth = contentDimensions.contentWidth
        scrollEvent.contentTop = contentDimensions.contentTop
        scrollEvent.contentBottom = contentDimensions.contentBottom

        // initial imgs refresh
        this.imgsUpdate()
      },

      // -------- For Refresher Component --------
      /**
       * 获取scrollElement元素的Dom
       * @private
       * */
      getScrollElement () {
        return this.scrollElement
      },

      /**
       * 滚动结束的事件回调
       * @param {function} callback - 过渡结束的回调, 回调传参TransitionEvent
       * @private
       */
      onScrollElementTransitionEnd (callback) {
        transitionEnd(this.scrollElement, callback)
      },

      /**
       * 在scrollElement上设置属性
       * @param {string} prop - 属性名称
       * @param {any} val     - 属性值
       * @private
       */
      setScrollElementStyle (prop, val) {
        if (this.scrollElement) {
          this.$nextTick(() => {
            (this.scrollElement.style)[prop] = val
          })
        }
      },

      // -------- For Img Component --------
      /**
       * @param {object} img - Img组件的实例
       * @private
       */
      addImg (img) {
        this._imgs.push(img)
      },
      /**
       *  @param {object} img - Img组件的实例
       *  @private
       */
      removeImg (img) {
        removeArrayItem(this._imgs, img)
      },
      /**
       * Img组件更新
       * @private
       */
      imgsUpdate () {
        if (this._scroll.initialized && this._imgs.length && this.isImgsUpdatable()) {
          let contentDimensions = this.getContentDimensions()
          this.$nextTick(() => {
            updateImgs(this._imgs, contentDimensions.scrollTop, contentDimensions.contentHeight, contentDimensions.directionY, this._imgReqBfr, this._imgRndBfr)
          })
        }
      },

      /**
       * @private
       * */
      isImgsUpdatable () {
        // 当滚动不是太快的时候, Img组件更新才被允许, 这个速度由this.imgVelMax控制
        return Math.abs(this._scroll.ev.velocityY) < this._imgVelMax
      },

      /**
       * 记录滚动位置
       * @private
       * */
      recordScrollPosition (ev) {
        if (this.recordPosition && this.$route) {
          // vm-scroll-position-${id} -> 位置
          let id = this.$route.name || this.$route.path
          window.sessionStorage.setItem(`vm-scroll-position-${id}`, ev.scrollTop)
        }
      },

      /**
       * 滚动到记录到页面滚动位置
       * @private
       * */
      toRecordScrollPosition () {
        if (this.recordPosition && this.$route) {
          let id = this.$route.name || this.$route.path
          if (this.$history.getDirection() === 'backward') {
            let scrollPositionStr = window.sessionStorage.getItem(`vm-scroll-position-${id}`)
            let scrollPosition = parseInt(scrollPositionStr)
            this.$nextTick(() => {
              this.scrollTo(0, scrollPosition, 0)
            })
          } else {
            window.sessionStorage.removeItem(`vm-scroll-position-${id}`)
          }
        }
      }
    },
    created () {
      // 页面进入前完成非DOM操作部分
      this._imgReqBfr = this.$config && this.$config.getNumber('imgRequestBuffer', 1400)
      this._imgRndBfr = this.$config && this.$config.getNumber('imgRenderBuffer', 600)
      this._imgVelMax = this.$config && this.$config.getNumber('imgVelocityMax', 3)
      this._scroll = new ScrollView()
      this._imgs = []
    },
    mounted () {
      this.init()

      // 为slot="fixed"/slot="fixedTop"/slot="fixedBottom"的沟槽设定属性
      if (this.$slots && this.$slots['fixed']) {
        this.$slots['fixed'].forEach(function (item) {
          item.elm.setAttribute('fixed', '')
        })
      }
      if (this.$slots && this.$slots['fixedTop']) {
        this.$slots['fixedTop'].forEach(function (item) {
          item.elm.setAttribute('fixed', '')
          item.elm.setAttribute('fixedTop', '')
        })
      }
      if (this.$slots && this.$slots['fixedBottom']) {
        this.$slots['fixedBottom'].forEach(function (item) {
          item.elm.setAttribute('fixed', '')
          item.elm.setAttribute('fixedBottom', '')
        })
      }

      this.toRecordScrollPosition()
    },
    activated () {
      this.toRecordScrollPosition()
    },
    deactivate () {},
    destroyed () {
      this._scroll.destroy()
    }
  }

  /**
   * 移除array中的某个item
   * @param {any} array
   * @param {any} item
   * @private
   */
  function removeArrayItem (array, item) {
    const index = array.indexOf(item)
    // ~index => index*(-1)-1
    // ~-1 => 0
    return ~index && array.splice(index, 1)
  }

  /**
   * 添加px后缀
   * @param {string} val
   * @return {string}
   * @private
   * */
  function cssFormat (val) {
    return (val > 0 ? val + 'px' : '')
  }

  /**
   *
   * 对两个img组件根据top排序
   * @param {object} a - Img组件实例
   * @param {object} b - Img组件实例
   * @return {number}
   * @private
   * */
  function sortTopToBottom (a, b) {
    if (a.top < b.top) {
      return -1
    }
    if (a.top > b.top) {
      return 1
    }
    return 0
  }

  /**
   * 更新img
   * @param {Img[]} imgs
   * @param {number} viewableTop
   * @param {number} contentHeight
   * @param {string} scrollDirectionY
   * @param {number} requestableBuffer
   * @param {number} renderableBuffer
   * @private
   * */
  function updateImgs (imgs, viewableTop, contentHeight, scrollDirectionY, requestableBuffer, renderableBuffer) {
    // ok, so it's time to see which images, if any, should be requested and rendered
    // ultimately, if we're scrolling fast then don't bother requesting or rendering
    // when scrolling is done, then it needs to do a check to see which images are
    // important to request and render, and which image requests should be aborted.
    // Additionally, images which are not near the viewable area should not be
    // rendered at all in order to save browser resources.
    const viewableBottom = (viewableTop + contentHeight)
    const priority1 = [] // Img[]
    const priority2 = [] // Img[]
    let img // 每个Img的实例;

    // all images should be paused
    for (var i = 0, ilen = imgs.length; i < ilen; i++) {
      img = imgs[i]

      if (scrollDirectionY === 'up') {
        // scrolling up
        if (img.getTop() < viewableBottom && img.getBottom() > viewableTop - renderableBuffer) {
          // 可视区向上移动, 图片在可是区域或者在可视区域的上面一点, 按照滚动方向即将要看到图片
          img.canRequest = img.canRender = true
          priority1.push(img)
          continue
        }

        if (img.getBottom() <= viewableTop && img.getBottom() > viewableTop - requestableBuffer) {
          // 可视区向上移动, 图片在可视区的上面, 还未进入, 但是需要提前发出图片请求
          img.canRequest = true
          img.canRender = false
          priority2.push(img)
          continue
        }

        if (img.getTop() >= viewableBottom && img.getTop() < viewableBottom + renderableBuffer) {
          // 可视区向上移动, 图片在可视区的下面, 所以按照这个方向移动, 是不会再看到图片,
          // 但是图片还是可能在renderable area, 故不需要reset
          img.canRequest = img.canRender = false
          continue
        }
      } else {
        // scrolling down
        if (img.getBottom() > viewableTop && img.getTop() < viewableBottom + renderableBuffer) {
          // 可视区向下移动,  图片在可是区域或者在可视区域的下面一点, 按照滚动方向即将要看到图片
          img.canRequest = img.canRender = true
          priority1.push(img)
          continue
        }

        if (img.getTop() >= viewableBottom && img.getTop() < viewableBottom + requestableBuffer) {
          // 可视区向下移动, 图片在可视区的下面, 还未进入, 但是需要提前发出图片请求
          img.canRequest = true
          img.canRender = false
          priority2.push(img)
          continue
        }

        if (img.getBottom() <= viewableTop && img.getBottom() > viewableTop - renderableBuffer) {
          // 可视区向下移动, 图片在可视区的上面, 所以按照这个方向移动, 是不会再看到图片,
          // 但是图片还是可能在renderable area, 故不需要reset
          img.canRequest = img.canRender = false
          continue
        }
      }

      img.canRequest = img.canRender = false
      img.reset()
    }

    // update all imgs which are viewable
    priority1.sort(sortTopToBottom).forEach(i => i.update())

    if (scrollDirectionY === 'up') {
      // scrolling up
      priority2.sort(sortTopToBottom).reverse().forEach(i => i.update())
    } else {
      // scrolling down
      priority2.sort(sortTopToBottom).forEach(i => i.update())
    }
  }
</script>
