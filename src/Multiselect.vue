<template>
  <div
    :tabindex="searchable ? -1 : tabindex"
    :class="{ 'multiselect--active': isOpen, 'multiselect--disabled': disabled, 'multiselect--above': isAbove }"
    @focus="activate()"
    @blur="searchable ? false : deactivate()"
    @keydown.self.down.prevent="pointerForward()"
    @keydown.self.up.prevent="pointerBackward()"
    @keypress.enter.tab.stop.self="addPointerElement($event)"
    @keyup.esc="deactivate()"
    class="multiselect">
      <slot name="caret" :toggle="toggle">
        <div @mousedown.prevent.stop="toggle()" class="multiselect__select"></div>
      </slot>
      <slot name="clear" :search="search"></slot>
      <div ref="tags" class="multiselect__tags">
        <slot
          name="selection"
          :search="search"
          :remove="removeElement"
          :values="visibleValues"
          :is-open="isOpen"
        >
          <div class="multiselect__tags-wrap" v-show="visibleValues.length > 0">
            <template v-for="(option, index) of visibleValues" @mousedown.prevent>
              <slot name="tag" :option="option" :search="search" :remove="removeElement">
                <span class="multiselect__tag" :key="index">
                  <span v-text="getOptionLabel(option)"></span>
                  <i aria-hidden="true" tabindex="1" @keypress.enter.prevent="removeElement(option)"  @mousedown.prevent="removeElement(option)" class="multiselect__tag-icon"></i>
                </span>
              </slot>
            </template>
          </div>
          <template v-if="internalValue && internalValue.length > limit">
            <slot name="limit">
              <strong class="multiselect__strong" v-text="limitText(internalValue.length - limit)"/>
            </slot>
          </template>
        </slot>
        <transition name="multiselect__loading">
          <slot name="loading">
            <div v-show="loading" class="multiselect__spinner"/>
          </slot>
        </transition>
        <input
          ref="search"
          v-if="searchable"
          :name="name"
          :id="id"
          type="text"
          autocomplete="nope"
          :placeholder="placeholder"
          :style="inputStyle"
          :value="search"
          :disabled="disabled"
          :tabindex="tabindex"
          @input="updateSearch($event.target.value)"
          @focus.prevent="activate()"
          @blur.prevent="deactivate()"
          @keyup.esc="deactivate()"
          @keydown.down.prevent="pointerForward()"
          @keydown.up.prevent="pointerBackward()"
          @keypress.enter.prevent.stop.self="addPointerElement($event)"
          @keydown.delete.stop="removeLastElement()"
          class="multiselect__input"/>
        <span
          v-if="isSingleLabelVisible"
          class="multiselect__single"
          @mousedown.prevent="toggle">
          <slot name="singleLabel" :option="singleValue">
            <template>{{ currentOptionLabel }}</template>
          </slot>
        </span>
        <span
          v-if="isPlaceholderVisible"
          class="multiselect__placeholder"
          @mousedown.prevent="toggle">
          <slot name="placeholder">
              {{ placeholder }}
          </slot>
        </span>
      </div>
      <transition name="multiselect">
        <div
          class="multiselect__content-wrapper"
          v-show="isOpen"
          @focus="activate"
          tabindex="-1"
          @mousedown.prevent
          :style="{ maxHeight: optimizedHeight + 'px' }"
          ref="list">
          <ul class="multiselect__content" :style="contentStyle">
            <slot name="beforeList"></slot>
            <li v-if="multiple && max === internalValue.length">
              <span class="multiselect__option">
                <slot name="maxElements">Maximum of {{ max }} options selected. First remove a selected option to select another.</slot>
              </span>
            </li>
            <template v-if="!max || internalValue.length < max">
              <li class="multiselect__element" v-for="(option, index) of filteredOptions" :key="index">
                <span
                  v-if="!(option && (option.$isLabel || option.$isDisabled))"
                  :class="optionHighlight(index, option)"
                  @click.stop="select(option)"
                  @mouseenter.self="pointerSet(index)"
                  :data-select="option && option.isTag ? tagPlaceholder : selectLabelText"
                  :data-selected="selectedLabelText"
                  :data-deselect="deselectLabelText"
                  class="multiselect__option">
                    <slot name="option" :option="option" :search="search">
                      <span>{{ getOptionLabel(option) }}</span>
                    </slot>
                </span>
                <span
                  v-if="option && (option.$isLabel || option.$isDisabled)"
                  :data-select="groupSelect && selectGroupLabelText"
                  :data-deselect="groupSelect && deselectGroupLabelText"
                  :class="groupHighlight(index, option)"
                  @mouseenter.self="groupSelect && pointerSet(index)"
                  @mousedown.prevent="selectGroup(option)"
                  class="multiselect__option">
                    <slot name="option" :option="option" :search="search">
                      <span>{{ getOptionLabel(option) }}</span>
                    </slot>
                </span>
              </li>
            </template>
            <li v-show="showNoResults && (filteredOptions.length === 0 && search && !loading)">
              <span class="multiselect__option">
                <slot name="noResult">No elements found. Consider changing the search query.</slot>
              </span>
            </li>
            <li v-show="showNoOptions && (options.length === 0 && !search && !loading)">
              <span class="multiselect__option">
                <slot name="noOptions">List is empty.</slot>
              </span>
            </li>
            <slot name="afterList"></slot>
          </ul>
        </div>
      </transition>
  </div>
</template>

<script>
import multiselectMixin from './multiselectMixin'
import pointerMixin from './pointerMixin'

export default {
  name: 'vue-multiselect',
  mixins: [multiselectMixin, pointerMixin],
  props: {
    /**
     * name attribute to match optional label element
     * @default ''
     * @type {String}
     */
    name: {
      type: String,
      default: ''
    },
    /**
     * String to show when pointing to an option
     * @default 'Press enter to select'
     * @type {String}
     */
    selectLabel: {
      type: String,
      default: 'Press enter to select'
    },
    /**
     * String to show when pointing to an option
     * @default 'Press enter to select'
     * @type {String}
     */
    selectGroupLabel: {
      type: String,
      default: 'Press enter to select group'
    },
    /**
     * String to show next to selected option
     * @default 'Selected'
     * @type {String}
     */
    selectedLabel: {
      type: String,
      default: 'Selected'
    },
    /**
     * String to show when pointing to an already selected option
     * @default 'Press enter to remove'
     * @type {String}
     */
    deselectLabel: {
      type: String,
      default: 'Press enter to remove'
    },
    /**
     * String to show when pointing to an already selected option
     * @default 'Press enter to remove'
     * @type {String}
     */
    deselectGroupLabel: {
      type: String,
      default: 'Press enter to deselect group'
    },
    /**
     * Decide whether to show pointer labels
     * @default true
     * @type {Boolean}
     */
    showLabels: {
      type: Boolean,
      default: true
    },
    /**
     * Limit the display of selected options. The rest will be hidden within the limitText string.
     * @default 99999
     * @type {Integer}
     */
    limit: {
      type: Number,
      default: 99999
    },
    /**
     * Sets maxHeight style value of the dropdown
     * @default 300
     * @type {Integer}
     */
    maxHeight: {
      type: Number,
      default: 300
    },
    /**
     * Function that process the message shown when selected
     * elements pass the defined limit.
     * @default 'and * more'
     * @param {Int} count Number of elements more than limit
     * @type {Function}
     */
    limitText: {
      type: Function,
      default: count => `and ${count} more`
    },
    /**
     * Set true to trigger the loading spinner.
     * @default False
     * @type {Boolean}
     */
    loading: {
      type: Boolean,
      default: false
    },
    /**
     * Disables the multiselect if true.
     * @default false
     * @type {Boolean}
     */
    disabled: {
      type: Boolean,
      default: false
    },
    /**
     * Fixed opening direction
     * @default ''
     * @type {String}
     */
    openDirection: {
      type: String,
      default: ''
    },
    /**
     * Shows slot with message about empty options
     * @default true
     * @type {Boolean}
     */
    showNoOptions: {
      type: Boolean,
      default: true
    },
    showNoResults: {
      type: Boolean,
      default: true
    },
    tabindex: {
      type: Number,
      default: 0
    }
  },
  computed: {
    isSingleLabelVisible () {
      return (
        (this.singleValue || this.singleValue === 0) &&
        (!this.isOpen || !this.searchable) &&
        !this.visibleValues.length
      )
    },
    isPlaceholderVisible () {
      return !this.internalValue.length && (!this.searchable || !this.isOpen)
    },
    visibleValues () {
      return this.multiple ? this.internalValue.slice(0, this.limit) : []
    },
    singleValue () {
      return this.internalValue[0]
    },
    deselectLabelText () {
      return this.showLabels ? this.deselectLabel : ''
    },
    deselectGroupLabelText () {
      return this.showLabels ? this.deselectGroupLabel : ''
    },
    selectLabelText () {
      return this.showLabels ? this.selectLabel : ''
    },
    selectGroupLabelText () {
      return this.showLabels ? this.selectGroupLabel : ''
    },
    selectedLabelText () {
      return this.showLabels ? this.selectedLabel : ''
    },
    inputStyle () {
      if (
        this.searchable ||
        (this.multiple && this.value && this.value.length)
      ) {
        // Hide input by setting the width to 0 allowing it to receive focus
        return this.isOpen
          ? { width: '100%' }
          : { width: '0', position: 'absolute', padding: '0' }
      }
    },
    contentStyle () {
      return this.options.length
        ? { display: 'inline-block' }
        : { display: 'block' }
    },
    isAbove () {
      if (this.openDirection === 'above' || this.openDirection === 'top') {
        return true
      } else if (
        this.openDirection === 'below' ||
        this.openDirection === 'bottom'
      ) {
        return false
      } else {
        return this.preferredOpenDirection === 'above'
      }
    },
    showSearchInput () {
      return (
        this.searchable &&
        (this.hasSingleSelectedSlot &&
        (this.visibleSingleValue || this.visibleSingleValue === 0)
          ? this.isOpen
          : true)
      )
    }
  }
}
</script>

<style>
  fieldset[disabled] .multiselect {
    pointer-events: none
  }

  .multiselect__spinner {
    position: absolute;
    right: 1px;
    top: 1px;
    width: 48px;
    height: 35px;
    background: #fff;
    display: block
  }

  .multiselect__spinner:after, .multiselect__spinner:before {
    position: absolute;
    content: "";
    top: 50%;
    left: 50%;
    margin: -8px 0 0 -8px;
    width: 16px;
    height: 16px;
    border-radius: 100%;
    border: 2px solid transparent;
    border-top-color: #716aca;
    box-shadow: 0 0 0 1px transparent
  }

  .multiselect__spinner:before {
    animation: spinning 2.4s cubic-bezier(.41, .26, .2, .62);
    animation-iteration-count: infinite
  }

  .multiselect__spinner:after {
    animation: spinning 2.4s cubic-bezier(.51, .09, .21, .8);
    animation-iteration-count: infinite
  }

  .multiselect__loading-enter-active, .multiselect__loading-leave-active {
    transition: opacity .4s ease-in-out;
    opacity: 1
  }

  .multiselect__loading-enter, .multiselect__loading-leave-active {
    opacity: 0
  }

  .multiselect, .multiselect__input, .multiselect__single {
    font-family: inherit;
    font-size: 16px;
    -ms-touch-action: manipulation;
    touch-action: manipulation
  }

  .multiselect {
    box-sizing: content-box;
    display: block;
    position: relative;
    width: 100%;
    min-height: 40px;
    text-align: left;
    color: #35495e
  }

  .multiselect * {
    box-sizing: border-box
  }

  .multiselect:focus {
    outline: none
  }

  .multiselect--disabled {
    background: #ededed;
    pointer-events: none;
    opacity: .6
  }

  .multiselect--active {
    z-index: 50
  }

  .multiselect--active:not(.multiselect--above) .multiselect__current, .multiselect--active:not(.multiselect--above) .multiselect__input, .multiselect--active:not(.multiselect--above) .multiselect__tags {
    border-bottom-left-radius: 0;
    border-bottom-right-radius: 0
  }

  .multiselect--active .multiselect__select {
    transform: rotate(180deg)
  }

  .multiselect--above.multiselect--active .multiselect__current, .multiselect--above.multiselect--active .multiselect__input, .multiselect--above.multiselect--active .multiselect__tags {
    border-top-left-radius: 0;
    border-top-right-radius: 0
  }

  .multiselect__input, .multiselect__single {
    position: relative;
    display: inline-block;
    min-height: 20px;
    line-height: 20px;
    border: none;
    border-radius: 5px;
    background: #fff;
    padding: 0 0 0 5px;
    width: 100%;
    transition: border .1s ease;
    box-sizing: border-box;
    margin-bottom: 8px;
    vertical-align: top
  }

  .multiselect__input:-ms-input-placeholder {
    color: #35495e
  }

  .multiselect__input::placeholder {
    color: #35495e
  }

  .multiselect__tag ~ .multiselect__input, .multiselect__tag ~ .multiselect__single {
    width: auto
  }

  .multiselect__input:hover, .multiselect__single:hover {
    border-color: #cfcfcf
  }

  .multiselect__input:focus, .multiselect__single:focus {
    border-color: #a8a8a8;
    outline: none
  }

  .multiselect__single {
    padding-left: 5px;
    margin-bottom: 8px
  }

  .multiselect__tags-wrap {
    display: inline
  }

  .multiselect__tags {
    min-height: 40px;
    display: block;
    padding: 8px 40px 0 8px;
    border-radius: 5px;
    border: 1px solid #e8e8e8;
    background: #fff;
    font-size: 14px
  }

  .multiselect__tag {
    position: relative;
    display: inline-block;
    padding: 4px 26px 4px 10px;
    border-radius: 5px;
    margin-right: 10px;
    color: #fff;
    line-height: 1;
    background: #716aca;
    margin-bottom: 5px;
    white-space: nowrap;
    overflow: hidden;
    max-width: 100%;
    text-overflow: ellipsis
  }

  .multiselect__tag-icon {
    cursor: pointer;
    margin-left: 7px;
    position: absolute;
    right: 0;
    top: 0;
    bottom: 0;
    font-weight: 700;
    font-style: normal;
    width: 22px;
    text-align: center;
    line-height: 22px;
    transition: all .2s ease;
    border-radius: 5px
  }

  .multiselect__tag-icon:after {
    content: "\D7";
    color: #ffffff;
    font-size: 14px
  }

  .multiselect__tag-icon:focus, .multiselect__tag-icon:hover {
    background: #f4516c
  }

  .multiselect__tag-icon:focus:after, .multiselect__tag-icon:hover:after {
    color: #fff
  }

  .multiselect__current {
    min-height: 40px;
    overflow: hidden;
    padding: 8px 30px 0 12px;
    white-space: nowrap;
    border-radius: 5px;
    border: 1px solid #e8e8e8
  }

  .multiselect__current, .multiselect__select {
    line-height: 16px;
    box-sizing: border-box;
    display: block;
    margin: 0;
    text-decoration: none;
    cursor: pointer
  }

  .multiselect__select {
    position: absolute;
    width: 40px;
    height: 38px;
    right: 1px;
    top: 1px;
    padding: 4px 8px;
    text-align: center;
    transition: transform .2s ease
  }

  .multiselect__select:before {
    position: relative;
    right: 0;
    top: 65%;
    color: #999;
    margin-top: 4px;
    border-color: #999 transparent transparent;
    border-style: solid;
    border-width: 5px 5px 0;
    content: ""
  }

  .multiselect__placeholder {
    color: #adadad;
    display: inline-block;
    margin-bottom: 10px;
    padding-top: 2px
  }

  .multiselect--active .multiselect__placeholder {
    display: none
  }

  .multiselect__content-wrapper {
    position: absolute;
    display: block;
    background: #fff;
    width: 100%;
    max-height: 240px;
    overflow: auto;
    border: 1px solid #e8e8e8;
    border-top: none;
    border-bottom-left-radius: 5px;
    border-bottom-right-radius: 5px;
    z-index: 50;
    -webkit-overflow-scrolling: touch
  }

  .multiselect__content {
    list-style: none;
    display: inline-block;
    padding: 0;
    margin: 0;
    min-width: 100%;
    vertical-align: top
  }

  .multiselect--above .multiselect__content-wrapper {
    bottom: 100%;
    border-bottom-left-radius: 0;
    border-bottom-right-radius: 0;
    border-top-left-radius: 5px;
    border-top-right-radius: 5px;
    border-bottom: none;
    border-top: 1px solid #e8e8e8
  }

  .multiselect__content::webkit-scrollbar {
    display: none
  }

  .multiselect__element {
    display: block
  }

  .multiselect__option {
    display: block;
    padding: 12px;
    min-height: 40px;
    line-height: 16px;
    text-decoration: none;
    text-transform: none;
    vertical-align: middle;
    position: relative;
    cursor: pointer;
    white-space: nowrap
  }

  .multiselect__option:after {
    top: 0;
    right: 0;
    position: absolute;
    line-height: 40px;
    padding-right: 12px;
    padding-left: 20px;
    font-size: 13px
  }

  .multiselect__option--highlight {
    background: #716aca;
    outline: none;
    color: #fff
  }

  .multiselect__option--highlight:after {
    content: attr(data-select);
    background: #716aca;
    color: #fff
  }

  .multiselect__option--selected {
    background: #f3f3f3;
    color: #35495e;
    font-weight: 700
  }

  .multiselect__option--selected:after {
    content: attr(data-selected);
    color: silver
  }

  .multiselect__option--selected.multiselect__option--highlight {
    background: #f4516c;
    color: #fff
  }

  .multiselect__option--selected.multiselect__option--highlight:after {
    background: #f4516c;
    content: attr(data-deselect);
    color: #fff
  }

  .multiselect--disabled .multiselect__current, .multiselect--disabled .multiselect__select {
    background: #ededed;
    color: #a6a6a6
  }

  .multiselect__option--disabled {
    background: #ededed !important;
    color: #a6a6a6 !important;
    cursor: text;
    pointer-events: none
  }

  .multiselect__option--group {
    background: #ededed;
    color: #35495e
  }

  .multiselect__option--group.multiselect__option--highlight {
    background: #35495e;
    color: #fff
  }

  .multiselect__option--group.multiselect__option--highlight:after {
    background: #35495e
  }

  .multiselect__option--disabled.multiselect__option--highlight {
    background: #dedede
  }

  .multiselect__option--group-selected.multiselect__option--highlight {
    background: #f4516c;
    color: #fff
  }

  .multiselect__option--group-selected.multiselect__option--highlight:after {
    background: #f4516c;
    content: attr(data-deselect);
    color: #fff
  }

  .multiselect-enter-active, .multiselect-leave-active {
    transition: all .15s ease
  }

  .multiselect-enter, .multiselect-leave-active {
    opacity: 0
  }

  .multiselect__strong {
    margin-bottom: 8px;
    line-height: 20px;
    display: inline-block;
    vertical-align: top
  }

  [dir=rtl] .multiselect {
    text-align: right
  }

  [dir=rtl] .multiselect__select {
    right: auto;
    left: 1px
  }

  [dir=rtl] .multiselect__tags {
    padding: 8px 8px 0 40px
  }

  [dir=rtl] .multiselect__content {
    text-align: right
  }

  [dir=rtl] .multiselect__option:after {
    right: auto;
    left: 0
  }

  [dir=rtl] .multiselect__clear {
    right: auto;
    left: 12px
  }

  [dir=rtl] .multiselect__spinner {
    right: auto;
    left: 1px
  }

  @keyframes spinning {
    0% {
      transform: rotate(0)
    }
    to {
      transform: rotate(2turn)
    }
  }
</style>
