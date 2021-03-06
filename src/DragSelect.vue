<template>
  <div class="vue-drag-select" @mousedown="onMouseDown">
    <slot :selectedItems="selectedItems" />
    <div v-if="mouseDown" class="vue-drag-select-box"
      :style="selectionBoxStyling"></div>
  </div>
</template>

<script>
  export default {
    name: 'vue-drag-select',
    props: {
      selectableItems: {
        type: [Array, Function],
        required: false
      }
    },
    data () {
      return {
        mouseDown: false,
        changedItems: false,
        startPoint: null,
        endPoint: null,
        selectedItems: []
      }
    },
    computed: {
      selectionBox () {
        // Only set styling when necessary
        if (!this.mouseDown || !this.startPoint || !this.endPoint) return {}

        const clientRect = this.$el.getBoundingClientRect()
        const scroll = this.getScroll()

        // Calculate position and dimensions of the selection box
        const left = Math.min(this.startPoint.x, this.endPoint.x) - clientRect.left - scroll.x
        const top = Math.min(this.startPoint.y, this.endPoint.y) - clientRect.top - scroll.y
        const width = Math.abs(this.startPoint.x - this.endPoint.x)
        const height = Math.abs(this.startPoint.y - this.endPoint.y)

        // Return the styles to be applied
        return {
          left,
          top,
          width,
          height
        }
      },
      selectionBoxStyling () {
        // Only set styling when necessary
        if (!this.mouseDown || !this.startPoint || !this.endPoint) return {}

        const { left, top, width, height } = this.selectionBox

        // Return the styles to be applied
        return {
          left: `${left}px`,
          top: `${top}px`,
          width: `${width}px`,
          height: `${height}px`
        }
      },
      items () {
        const t = typeof this.selectableItems
        switch (t) {
          case 'function':
            return this.selectableItems()
            break
          case 'array':
            return this.selectableItems
            break
          case 'undefined':
            return this.$children.length
                    ? this.$children
                    : this.$el.children
          default:
            throw `unsupported type '${t}' for property 'selectableItems'`
        }
      }
    },
    watch: {
      selectedItems (val) {
        this.$emit('change', val)
      }
    },
    methods: {
      getScroll () {
        // If we're on the server, default to 0,0
        if (typeof document === 'undefined') {
          return {
            x: 0,
            y: 0
          }
        }

        return {
          x: this.$el.scrollLeft || document.body.scrollLeft || document.documentElement.scrollLeft,
          y: this.$el.scrollTop || document.body.scrollTop || document.documentElement.scrollTop
        }
      },
      onMouseDown (event) {
        // Ignore right clicks
        if (event.button === 2) return

        // Register begin point
        this.mouseDown = true
        this.startPoint = {
          x: event.pageX,
          y: event.pageY
        }

        // Start listening for mouse move and up events
        window.addEventListener('mousemove', this.onMouseMove)
        window.addEventListener('mouseup', this.onMouseUp)
      },
      onMouseMove (event) {
        // Update the end point position
        if (this.mouseDown) {
          this.endPoint = {
            x: event.pageX,
            y: event.pageY
          }

          if (this.items) {
            this.selectedItems = this.items.filter((item) => {
              return this.isItemSelected(item.$el || item)
            })
            this.changedItems = true
          }
        }
      },
      onMouseUp (event) {
        // Clean up event listeners
        window.removeEventListener('mousemove', this.onMouseMove)
        window.removeEventListener('mouseup', this.onMouseUp)

        if (this.changedItems)
          this.$emit('select', this.selectedItems)

        // Reset state
        this.mouseDown = false
        this.changedItems = false
        this.startPoint = null
        this.endPoint = null
      },
      isItemSelected (el) {
        const boxA = this.selectionBox
        const boxB = {
          top: el.offsetTop,
          left: el.offsetLeft,
          width: el.clientWidth,
          height: el.clientHeight
        }

        return !!(
          boxA.left <= boxB.left + boxB.width &&
          boxA.left + boxA.width >= boxB.left &&
          boxA.top <= boxB.top + boxB.height &&
          boxA.top + boxA.height >= boxB.top
        )
      }
    },
    mounted () {
      this.$children.forEach((child) => {
        child.$on('click', (event) => {
          const included = this.selectedItems.find((item) => {
            return child.$el === item.$el
          })

          if (included) {
            this.selectedItems = this.selectedItems.filter((item) => {
              return child.$el !== item.$el
            })
          } else {
            this.selectedItems.push(child)
          }
        })
      })
    },
    beforeDestroy () {
      // Remove event listeners
      window.removeEventListener('mousemove', this.onMouseMove)
      window.removeEventListener('mouseup', this.onMouseUp)

      this.$children.forEach((child) => {
        child.$off('click')
      })
    }
  }
</script>

<style>
  .vue-drag-select {
    position: relative;
    user-select: none;
  }

  .vue-drag-select-box {
    position: absolute;
    background: rgba(0, 162, 255, .4);
    z-index: 99;
  }
</style>
