<style>
  /* Disabled options */
  .v-select .dropdown-menu li .disabled {
    background-color: #f8f8f8;
    cursor: not-allowed;
  }

  /* Input Label */
  .v-select-label-span {
    float: left;
    width: 97%;
    display: inline-block;
    overflow: hidden;
    text-align: left;
    white-space: nowrap;
    text-overflow: ellipsis;
  }

  /* Dropdown Menu */
  .v-select .dropdown-menu {
    overflow-y: scroll;
  }
  .v-select .no-options {
    text-align: center;
    margin: 0 20px;
  }

  .v-select-option:hover {
    cursor: pointer;
  }

  /* Search Input */
  .v-select-search {
    padding-left: 10px;
    padding-right: 10px;
  }

  .v-select-search-input {
    width: 100%;
  }

  /* Dropdown Default Transition */
  .v-select-transition-enter-active,
  .v-select-transition-leave-active {
    transition: opacity .15s cubic-bezier(1.0, 0.5, 0.8, 1.0);
  }
  .v-select-transition-enter,
  .v-select-transition-leave-to {
    opacity: 0;
  }

  /* Processing Spinner */
  .v-select .loadingSpinner {
      margin: 0 auto;
      width: 70px;
      text-align: center;
  }

  .v-select .loadingSpinner > div {
      width: 18px;
      height: 18px;
      background-color: #343d41;
      border-radius: 100%;
      display: inline-block;
      -webkit-animation: vSelectBounceDelay 1.4s infinite ease-in-out;
      animation: vSelectBounceDelay 1.4s infinite ease-in-out;
      /* Prevent first frame from flickering when animation starts */
      -webkit-animation-fill-mode: both;
      animation-fill-mode: both;
  }

  .v-select .loadingSpinner .bounce1 {
      -webkit-animation-delay: -0.32s;
      animation-delay: -0.32s;
  }

  .v-select .loadingSpinner .bounce2 {
      -webkit-animation-delay: -0.16s;
      animation-delay: -0.16s;
  }

  @-webkit-keyframes vSelectBounceDelay {
      0%, 80%, 100% { -webkit-transform: scale(0.0) }
      40% { -webkit-transform: scale(1.0) }
  }

  @keyframes vSelectBounceDelay {
      0%, 80%, 100% {
          transform: scale(0.0);
          -webkit-transform: scale(0.0);
      } 40% {
          transform: scale(1.0);
          -webkit-transform: scale(1.0);
      }
  }
</style>

<template>
  <div v-click-outside="closeOnBlur" class="btn-block v-select" :class="dropdownClasses" :id="componentIdAttr">
    <button
      type="button"
      ref="toggle"
      @keyup.space="toggleDropdown"
      @keyup.enter="toggleDropdown"
      @click.prevent="toggleDropdown"
      @keyup.esc="onEscape"
      @keydown.down.prevent="tabDown"
      @keydown.tab="focusSearch"
      class="btn-block btn btn-default dropdown-toggle"
      :class="dropdownButtonClasses"
      :tabindex="tabindex + 1"
      :id="toggleIdAttr"
    >
      <span class="v-select-label-span" :id="toggleLabelIdAttr">{{ dropdownButtonLabel() }}</span>
      <span class="caret"></span>
      <template v-if="valueAsArray.length">
        <input
          type="hidden"
          class="v-select-selected-value"
          :class="{ required: required }"
          v-for="(option, index) in valueAsArray"
          v-bind:key="index"
          :id="getSelectedOptionIdAttr(index)"
          :name="getSelectedOptionNameAttr()"
          :value="getSelectedOptionValue(option)"
        >
      </template>
      <input
        type="hidden"
        class="v-select-selected-value"
        :class="{ required: required }"
        v-else-if="!valueAsArray.length"
        :id="getSelectedOptionIdAttr(0)"
        :name="getSelectedOptionNameAttr()"
        value=""
      >
    </button>

    <transition :name="transition">
      <ul :id="dropdownMenuIdAttr" ref="dropdownMenu" v-if="dropdownOpen" class="dropdown-menu" :style="{ 'max-height': maxHeight }">
        <li class="v-select-search">
          <input
            ref="search"
            v-model="search"
            @keyup.esc="onEscape"
            @keydown.down.prevent="tabDown"
            @keydown.up.prevent="toggleDropdown"
            @keydown.tab.shift.prevent="toggleDropdown"
            type="search"
            class="form-control v-select-search-input"
            :id="searchIdAttr"
            :placeholder="searchPlaceholder"
            :readonly="!searchable"
            aria-label="Search for option"
            :tabindex="tabindex + 2"
            v-focus
          >
        </li>
        <li role="separator" class="divider" v-if="valueAsArray.length || mutableLoading"></li>
        <slot name="spinner">
          <div :id="spinnerIdAttr" class="loadingSpinner" v-if="mutableLoading">
            <div class="bounce1"></div>
            <div class="bounce2"></div>
            <div class="bounce3"></div>
          </div>
        </slot>
        <li v-if="!mutableLoading" v-for="(option, index) in valueAsArray" v-bind:key="index" :class="{ active: isOptionSelected(option) }">
          <span v-if="isOptgroupOption(option)" tabindex="-1">
            {{ getOptionLabel(option) }}
          </span>
          <a
            v-if="!isOptgroupOption(option)"
            class="v-select-option v-select-option-selected"
            :id="getSelectedOptionAnchorIDAttr(index)"
            @click.prevent="select(option)"
            :tabindex="tabindex + 3 + index"
            @keyup.esc="onEscape"
            @keyup.enter.space.prevent="select(option)"
            @keydown.down.prevent="tabDown"
            @keydown.up.prevent="tabUp"
          >
            {{ getOptionLabel(option) }}
          </a>
        </li>
        <li v-if="!mutableLoading" role="separator" class="divider"></li>
        <li v-if="!mutableLoading" v-for="(option, index) in filteredOptions" v-bind:key="index" :class="{ 'dropdown-header': isOptgroupOption(option) }">
          <span v-if="isOptgroupOption(option)" tabindex="-1">
            {{ getOptionLabel(option) }}
          </span>
          <a
            v-else-if="isReadonlyOption(option)"
            class="disabled"
            :id="getFilteredOptionAnchorIDAttr(index)"
            :tabindex="-1"
          >
            {{ getOptionLabel(option) }}
          </a>
          <a
            v-else-if="!isOptgroupOption(option)"
            class="v-select-option"
            :id="getFilteredOptionAnchorIDAttr(index)"
            @click.prevent="select(option)"
            :tabindex="tabindex + 3 + valueAsArray.length + index"
            @keyup.esc="onEscape"
            @keyup.enter.space.prevent="select(option)"
            @keydown.down.prevent="tabDown"
            @keydown.up.prevent="tabUp"
          >
            {{ getOptionLabel(option) }}
          </a>
        </li>
        <li ref="noOptions" v-if="filteredOptions === undefined || !filteredOptions.length && !mutableLoading" :id="noOptionsIdAttr" class="no-options">
          <slot name="no-options">Sorry, no matching options.</slot>
        </li>
      </ul>
    </transition>
  </div>
</template>

<script type="text/babel">
  import ajax from '../mixins/ajax'
  import removeDiacritics from '../mixins/removeDiacritics'

  export default {
    mixins: [ajax, removeDiacritics],

    directives: {
      // v-focus
      focus: {
        inserted: function (el) {
          el.focus()
        }
      },

      // v-click-outside
      'click-outside': {
        bind: function(el, binding, vNode) {
          // Provided expression must evaluate to a function.
          if (typeof binding.value !== 'function') {
            const compName = vNode.context.name
            let warn = `[Vue-click-outside:] provided expression '${binding.expression}' is not a function, but has to be`
            if (compName) { warn += `Found in component '${compName}'` }

            console.warn(warn)
          }
          // Define Handler and cache it on the element
          const bubble = binding.modifiers.bubble
          const handler = (e) => {
            if (bubble || (!el.contains(e.target) && el !== e.target)) {
              binding.value(e)
            }
          }
          el.__vueClickOutside__ = handler

          // add Event Listeners
          document.addEventListener('click', handler)
        },

        unbind: function(el, binding) {
          // Remove Event Listeners
          document.removeEventListener('click', el.__vueClickOutside__)
          el.__vueClickOutside__ = null
        }
      }
    },

    props: {
      /**
       * Contains the currently selected value. Very similar to a
       * `value` attribute on an <input>. You can listen for changes
       * using 'change' event using v-on
       * @type {Object||String||null}
       */
      value: {
        default: null
      },

      /**
       * An array of strings or objects to be used as dropdown choices.
       * If you are using an array of objects, vue-select will look for
       * a `label` key (ex. [{label: 'This is Foo', value: 'foo'}]). A
       * custom label key can be set with the `label` prop.
       * @type {Object}
       */
      options: {
        type: Array,
        default() {
          return []
        },
      },

      /**
       * Dictate whether or not the element is required for input
       * validation purposes
       * @type {Boolean}
       */
      required: {
        type: Boolean,
        default: false
      },

      /**
       * Sets the max-height property on the dropdown list.
       * @deprecated
       * @type {String}
       */
      maxHeight: {
        type: String,
        default: '400px'
      },

      /**
       * Enable/disable filtering the options.
       * @type {Boolean}
       */
      searchable: {
        type: Boolean,
        default: true
      },

      /**
       * Equivalent to the `multiple` attribute on a `<select>` input.
       * @type {Object}
       */
      multiple: {
        type: Boolean,
        default: false
      },

      /**
       * Equivalent to the `placeholder` attribute on an `<input>`.
       * @type {Object}
       */
      placeholder: {
        type: String,
        default: ''
      },

      /**
       * Sets a Vue transition property on the `.dropdown-menu`. vue-select
       * does not include CSS for transitions, you'll need to add them yourself.
       * @type {String}
       */
      transition: {
        type: String,
        default: 'v-select-transition'
      },

      /**
       * Enables/disables clearing the search text when an option is selected.
       * @type {Boolean}
       */
      clearSearchOnSelect: {
        type: Boolean,
        default: true
      },

      /**
       * Close a dropdown when an option is chosen. Set to false to keep the dropdown
       * open (useful when combined with multi-select, for example)
       * @type {Boolean}
       */
      closeOnSelect: {
        type: Boolean,
        default: true
      },

      /**
       * Tells vue-select what key to use when generating option
       * labels when each `option` is an object.
       * @type {String}
       */
      label: {
        type: String,
        default: 'label'
      },

      /**
       * Tells vue-select what key to use when pulling option
       * values when each `option` is an object.
       * @type {String}
       */
      valueKey: {
        type: String,
        default: 'value'
      },

      /**
       * Callback to generate the label text. If {option}
       * is an object, returns option[this.label] by default.
       * @param  {Object || String} option
       * @return {String}
       */
      getOptionLabel: {
        type: Function,
        default(option) {
          if (typeof option === 'object') {
            if (this.label && option[this.label]) {
              return option[this.label]
            }
          }
          return option;
        }
      },

      /**
       * An optional callback function that is called each time the selected
       * value(s) change. When integrating with Vuex, use this callback to trigger
       * an action, rather than using :value.sync to retreive the selected value.
       * @type {Function}
       * @default {null}
       */
      onChange: {
        type: Function,
        default: function (val) {
          this.$emit('input', val)
        }
      },

      /**
       * An optional callback function that is called each time the dropdown
       * is opened.
       * @type {Function}
       * @default {null}
       */
      onOpen: {
        type: Function,
        default: function (flag) {
          this.$emit('open', flag)
        }
      },

      /**
       * Enable/disable creating options from searchInput.
       * @type {Boolean}
       */
      taggable: {
        type: Boolean,
        default: false
      },

      /**
       * When true, newly created tags will be added to
       * the options list.
       * @type {Boolean}
       */
      pushTags: {
        type: Boolean,
        default: false
      },

      /**
       * User defined function for adding Options
       * @type {Function}
       */
      createOption: {
        type: Function,
        default(newOption) {
          if (typeof this.mutableOptions[0] === 'object') {
            newOption = {[this.label]: newOption}
          }
          this.$emit('option:created', newOption)
          return newOption
        }
      },

      /**
       * When false, updating the options will not reset the select value
       * @type {Boolean}
       */
      resetOnOptionsChange: {
        type: Boolean,
        default: false
      },

      /**
       * Sets the name (and ID if not
       * present) of the input element.
       * @type {String}
       */
      inputName: {
        type: String,
        default: ''
      },

      /**
       * Sets the id of the input element.
       * @type {String}
       */
      inputId: {
        type: String,
        default() {
          return this.inputName.replace(/\]/g, '').replace(/\[/g, '_')
        }
      },

      /**
       * Enable the remove diacritics support
       * @type {Boolean}
       */
      removeDiacritics: {
        type: Boolean,
        default: false
      },

      /**
       * Contains the tabindex, which populates
       * other
       * @type {Number}
       */
      tabindex: {
        type: Number,
        default: 0
      },

      /**
       * Contains an array of values to mark as
       * readonly
       * @type {Array}
       */
      readonlyValues: {
        type: Array,
        default() {
          return []
        }
      },

      /**
       * Additional button classes
       * (e.g. btn-sm, btn-lg)
       * @type {String}
       */
      buttonClasses: {
        type: String,
        default: 'btn-md'
      },

      /**
       * Mark the button/dropdown as readonly
       * @type {Boolean}
       */
      readonly: {
        type: Boolean,
        default: false,
      },
    },

    data() {
      return {
        search: '',
        open: false,
        mutableValue: null,
        mutableOptions: []
      }
    },

    watch: {
      /**
       * When the value prop changes, update
			 * the internal mutableValue.
       * @param  {mixed} val
       * @return {void}
       */
      value(val) {
				this.mutableValue = val
      },

      /**
       * Maybe run the onChange callback.
       * @param  {string|object} val
       * @param  {string|object} old
       * @return {void}
       */
			mutableValue(val, old) {
        if (this.multiple) {
          this.onChange ? this.onChange(val) : null
        } else {
          this.onChange && val !== old ? this.onChange(val) : null
        }
      },

      /**
       * When options change, update
       * the internal mutableOptions.
       * @param  {array} val
       * @return {void}
       */
      options(val) {
        this.mutableOptions = val
      },

      /**
			 * Maybe reset the mutableValue
       * when mutableOptions change.
       * @return {[type]} [description]
       */
      mutableOptions() {
        if (!this.taggable && this.resetOnOptionsChange) {
					this.mutableValue = this.multiple ? [] : null
        }
      },

      /**
			 * Always reset the mutableValue when
       * the multiple prop changes.
       * @param  {Boolean} val
       * @return {void}
       */
      multiple(val) {
				this.mutableValue = val ? [] : null
      },
    },

    /**
     * Clone props into mutable values,
     * attach any event listeners.
     */
    created() {
			this.mutableValue = this.value
      this.mutableOptions = this.options.slice(0)
			this.mutableLoading = this.loading

      this.$on('option:created', this.maybePushTag)
    },

    methods: {
      /**
       * Open or close the dropdown and fire off the onOpen callback
       * @param  {Boolean} flag
       * @return {void}
       */
      toggleOpen(flag) {
        if (this.readonly) {
          this.open = false;
        } else {
          this.open = flag;
          this.onOpen(flag);
        }
      },

      /**
       * Select a given option.
       * @param  {Object|String} option
       * @return {void}
       */
      select(option) {
        if (this.isOptionSelected(option)) {
          this.deselect(option)
        } else {
          if (this.taggable && !this.optionExists(option)) {
            option = this.createOption(option)
          }

          if (this.multiple && !this.mutableValue) {
            this.mutableValue = [option]
          } else if (this.multiple) {
            this.mutableValue.push(option)
          } else {
            this.mutableValue = option
          }
        }

        this.onAfterSelect(option)
      },

      /**
       * De-select a given option.
       * @param  {Object|String} option
       * @return {void}
       */
      deselect(option) {
        if (this.multiple) {
          let ref = -1
          this.mutableValue.forEach((val) => {
            if (val === option || typeof val === 'object' && val[this.label] === option[this.label]) {
              ref = val
            }
          })
          let index = this.mutableValue.indexOf(ref)
          this.mutableValue.splice(index, 1)
        } else {
          this.mutableValue = null
        }
      },

      /**
       * Called from this.select after each selection.
       * @param  {Object|String} option
       * @return {void}
       */
      onAfterSelect(option) {
        if (this.closeOnSelect) {
          this.toggleOpen(!this.open)
          this.$refs.toggle.focus()
        }

        if (this.clearSearchOnSelect) {
          this.search = ''
        }
      },

      /**
       * Toggle the visibility of the dropdown menu.
       * @param  {Event} e
       * @return {void}
       */
      toggleDropdown(e) {
        if (this.open) {
          this.toggleOpen(false)
          if (this.clearSearchOnSelect) {
            this.search = ''
          }
          this.$refs.toggle.focus()
        } else {
          this.toggleOpen(true)
        }
      },

      /**
       * Close the dropdown on blur of elements within
       * @param  {Event} e
       * @return {void}
       */
      closeOnBlur(e) {
        this.toggleOpen(false)
        if (this.clearSearchOnSelect) {
          this.search = ''
        }
      },

      /**
       * Check if the given option is currently selected.
       * @param  {Object|String}  option
       * @return {Boolean}        True when selected | False otherwise
       */
      isOptionSelected(option) {
        if (this.multiple && this.mutableValue) {
          let selected = false
          this.mutableValue.forEach(opt => {
            if (typeof opt === 'object' && opt[this.label] === option[this.label]) {
              selected = true
            } else if (typeof opt === 'object' && opt[this.label] === option) {
              selected = true
            }
            else if (opt === option) {
              selected = true
            }
          })
          return selected
        }

        return this.mutableValue === option
      },

      /**
       * If there is any text in the search input, remove it.
       * Otherwise, blur the search input to close the dropdown.
       * @return {void}
       */
      onEscape() {
        this.search = ''
        this.toggleOpen(false)
      },

      /**
       * Determine if an option exists
       * within this.mutableOptions array.
       *
       * @param  {Object || String} option
       * @return {boolean}
       */
      optionExists(option) {
        let exists = false

        this.mutableOptions.forEach(opt => {
          if (typeof opt === 'object' && opt[this.label] === option) {
            exists = true
          } else if (opt === option) {
            exists = true
          }
        })

        return exists
      },

      /**
       * If push-tags is true, push the
       * given option to mutableOptions.
       *
       * @param  {Object || String} option
       * @return {void}
       */
      maybePushTag(option) {
        if (this.pushTags) {
          this.mutableOptions.push(option)
        }
      },

      /**
       * Check if the given option is an optgroup
       * @param  {Object|String}  option
       * @return {Boolean}        True if option value is an object | False otherwise
       */
      isOptgroupOption(option) {
        return option['optgroup'] === true
      },

      /**
       * Toggle the visibility of the dropdown menu
       * or focus the search input if open
       * @param  {Event} e
       * @return {void}
       */
      focusSearch(e) {
        if (this.open) {
          e.preventDefault()
          this.$refs.search.focus()
        }
      },

      /**
       * Simulate a tab key press to navigate the dropdown
       * @param  {Event} e
       * @return {void}
       */
      tabDown(e) {
        if (this.$refs.toggle === e.target) {
          if (this.open) {
            this.$refs.search.focus()
          } else {
            this.toggleOpen(true)
            return;
          }
        }

        const tabIndex = parseInt(document.activeElement.getAttribute('tabindex'), 10)
        const nodes = this.$el.querySelectorAll('.v-select-option')
        const lastTabIndex = parseInt(nodes[nodes.length - 1].getAttribute('tabindex'), 10)
        if (tabIndex < lastTabIndex) {
          let nextIndex = 1
          while (this.$el.querySelectorAll(`.v-select-option[tabindex="${tabIndex + nextIndex}"]`)[0] === undefined) {
            nextIndex += 1
          }
          this.$el.querySelectorAll(`.v-select-option[tabindex="${tabIndex + nextIndex}"]`)[0].focus()
        }
      },

      /**
       * Simulate a shift+tab key press to navigate the dropdown
       * @param  {Event} e
       * @return {void}
       */
      tabUp(e) {
        const tabIndex = parseInt(document.activeElement.getAttribute('tabindex'), 10)
        const nodes = this.$el.querySelectorAll('.v-select-option')
        const firstTabIndex = parseInt(nodes[0].getAttribute('tabindex'), 10)
        let pSibling = document.activeElement.parentNode.previousSibling
        if (tabIndex > firstTabIndex) {
          let prevIndex = 1
          while (this.$el.querySelectorAll(`.v-select-option[tabindex="${tabIndex - prevIndex}"]`)[0] === undefined) {
            prevIndex += 1
          }
          this.$el.querySelectorAll(`.v-select-option[tabindex="${tabIndex - prevIndex}"]`)[0].focus()
        } else if (this.$refs.search === e.target) {
          this.$refs.toggle.focus()
        } else {
          this.$refs.search.focus()
        }
      },

      /**
       * Get the label for the dropdown button
       */
      dropdownButtonLabel() {
        if (!this.valueAsArray || this.valueAsArray.length === 0) {
          return "Select"
        }

        if (this.valueAsArray.length === 1) {
          if (typeof this.valueAsArray[0] === 'object' && this.valueAsArray[0].hasOwnProperty(this.label)) {
            return this.valueAsArray[0][this.label]
          } else {
            return this.valueAsArray[0]
          }
        }

        return 'Multiple'
      },

      /**
       * Get the anchor ID attr for a selected option link
       */
      getSelectedOptionAnchorIDAttr(index) {
        if (this.inputId === '') {
          return ''
        }
        return `${this.inputId}_selected_options_${index}`
      },

      /**
       * Get the anchor ID attr for a filtered option link
       */
      getFilteredOptionAnchorIDAttr(index) {
        if (this.inputId === '') {
          return ''
        }
        return `${this.inputId}_filtered_options_${index}`
      },

      /**
       * Get the hidden input ID attr for a selected option link
       */
      getSelectedOptionIdAttr(index) {
        if (this.inputId === '') {
          return ''
        }
        return `${this.inputId}_selected_options_input_${index}`
      },

      /**
       * Get the hidden input name attr for a selected option link
       */
      getSelectedOptionNameAttr(index) {
        if (this.inputName === '') {
          return ''
        }

        if (this.multiple) {
          return `${this.inputName}[]`
        }

        return `${this.inputName}`
      },

      /**
       * Get the hidden input name attr for a selected option link
       */
      getSelectedOptionValue(option) {
        if (typeof option === 'object' && option.hasOwnProperty(this.valueKey)) {
          return option[this.valueKey]
        } else if (typeof option === 'object' && !option.hasOwnProperty('value')) {
          return option['value']
        } else if (typeof option === 'object' && !option.hasOwnProperty(this.valueKey)) {
          return console.warn(`[vue-select warn]: Value key "option.${this.valueKey}" does not exist in options object.`)
        }
        return option
      },

      /**
       * Check if an option is readonly
       *
       * @param {Object} opt
       * @return {Boolean}
       */
      isReadonlyOption(option) {
        return JSON.stringify(this.readonlyValues).indexOf(JSON.stringify(option)) !== -1
      },

      /**
       * Determine is object or string is in the array
       * of selected values
       * @param {Object|String} item
       * @param {Array}  arr
       * @return {Boolean}
       */
       isItemInValuesArray(item, arr) {
         let selectedValues = JSON.parse(JSON.stringify(this.valueAsArray))

         if (typeof item === 'object' && item.hasOwnProperty(this.label) && item.hasOwnProperty(this.valueKey)) {
           return selectedValues.some((selected) => {
             if (typeof selected === 'object' && selected.hasOwnProperty(this.label) && selected.hasOwnProperty(this.valueKey)) {
               return item[this.label] === selected[this.label] && item[this.valueKey] === selected[this.valueKey]
             }
             return false
           })
         }

         if (selectedValues && selectedValues.includes(item)) {
           return true
         }

         return false
       }
    },

    computed: {

      /**
       * Classes to be output on .dropdown
       * @return {Object}
       */
      dropdownClasses() {
        return {
          open: this.dropdownOpen,
          single: !this.multiple,
          searching: this.searching,
          searchable: this.searchable,
          unsearchable: !this.searchable,
          loading: this.mutableLoading,
          required: this.required
        }
      },

      /**
       * Classes to be applied to the <button> element
       * @return {Object}
       */
      dropdownButtonClasses() {
        const classes = {};
        const classKeys = this.buttonClasses.split(' ');

        classKeys.forEach((key) => {
          classes[key] = true;
        });

        if (this.readonly) {
          classes.disabled = true;
        }

        return classes;
      },

      /**
       * If search text should clear on blur
       * @return {Boolean} True when single and clearSearchOnSelect
       */
      clearSearchOnBlur() {
        return this.clearSearchOnSelect && !this.multiple
      },

      /**
       * Return the current state of the
       * search input
       * @return {Boolean} True if non empty value
       */
      searching() {
        return !!this.search
      },

      /**
       * Return the current state of the
       * dropdown menu.
       * @return {Boolean} True if open
       */
      dropdownOpen() {
        return this.noDrop ? false : this.open
      },

      /**
       * Return the placeholder string if it's set
       * & there is no value selected.
       * @return {String} Placeholder text
       */
      searchPlaceholder() {
        return this.placeholder
      },

      /**
       * The currently displayed options, filtered
       * by the search elements value. If tagging
       * true, the search text will be prepended
       * if it doesn't already exist.
       *
       * @return {array}
       */
      filteredOptions() {
        // Flatten the structure to fall in line with Bootstrap compatibility
        // Unsorted options will be brought to the top of the list so they do
        // not fall under a header
        let flattenedOptions = []
        let unsortedOptions = []
        for (let prop in this.mutableOptions) {
          if (this.mutableOptions.hasOwnProperty(prop)) {
            let option = this.mutableOptions[prop]
            if (typeof option === 'object' && option.hasOwnProperty(this.label) && option.hasOwnProperty('options') && typeof option['options'] === 'object') {
              this.$set(option, 'optgroup', true)
              flattenedOptions.push(option)
              for (let p in option['options']) {
                if (option['options'].hasOwnProperty(p)) {
                  let opt = option['options'][p]

                  if (typeof opt === 'object' && opt.hasOwnProperty(this.label) && opt.hasOwnProperty(this.valueKey)) {
                    flattenedOptions.push(opt)
                  } else if (typeof option === 'object' && !option.hasOwnProperty(this.label)) {
                    return console.warn(`[vue-select warn]: Label key "option.${this.label}" does not exist in options object.\nhttp://sagalbot.github.io/vue-select/#ex-labels`)
                  } else {
                    flattenedOptions.push(opt)
                  }
                }
              }
            } else if (typeof option === 'object' && option.hasOwnProperty(this.label) && option.hasOwnProperty(this.valueKey)) {
              unsortedOptions.push(option)
            } else if (typeof option === 'object' && !option.hasOwnProperty(this.label)) {
              return console.warn(`[vue-select warn]: Label key "option.${this.label}" does not exist in options object.\nhttp://sagalbot.github.io/vue-select/#ex-labels`)
            } else {
              unsortedOptions.push(option)
            }
          }
        }
        const sortedOptions = unsortedOptions.concat(flattenedOptions)

        let selectedValues = JSON.parse(JSON.stringify(this.valueAsArray))
        let options = sortedOptions.filter((option) => {
          if (typeof option === 'object' && option.hasOwnProperty(this.label) && option.hasOwnProperty('options') && typeof option['options'] === 'object') {
            // Optgroups - filter the optgroup values
            let found = false
            for (let index in option['options']) {
              if (option['options'].hasOwnProperty(index)) {
                let opt = option['options'][index]
                if (typeof opt === 'object' && opt.hasOwnProperty(this.label)) {
                  if (this.removeDiacritics) {
                    found = this.removeDiacriticsFromString(opt[this.label]).toLowerCase().indexOf(this.search.toLowerCase()) > -1
                  } else {
                    found = opt[this.label].toLowerCase().indexOf(this.search.toLowerCase()) > -1
                  }
                } else if (typeof opt !== 'object') {
                  if (this.removeDiacritics) {
                    found = this.removeDiacriticsFromString(opt).toLowerCase().indexOf(this.search.toLowerCase()) > -1
                  } else {
                    found = opt.toLowerCase().indexOf(this.search.toLowerCase()) > -1
                  }
                }
                if (found) {
                  return found
                }
              }
            }
            return false
          } else if (typeof option === 'object' && option.hasOwnProperty(this.label)) {
            // Filter out those already selected, since we are looping them differently
            let o = JSON.parse(JSON.stringify(option))
            if (this.isItemInValuesArray(o, selectedValues)) {
              return false
            }
            if (this.removeDiacritics) {
              return this.removeDiacriticsFromString(option[this.label]).toLowerCase().indexOf(this.search.toLowerCase()) > -1
            }
            return option[this.label].toLowerCase().indexOf(this.search.toLowerCase()) > -1
          }

          // Filter out those already selected, since we are looping them differently
          if (this.isItemInValuesArray(option, selectedValues)) {
            return false
          }
          if (this.removeDiacritics) {
            return this.removeDiacriticsFromString(option).toLowerCase().indexOf(this.search.toLowerCase()) > -1
          }
          return option.toLowerCase().indexOf(this.search.toLowerCase()) > -1
        })
        if (this.taggable && this.search.length && !this.optionExists(this.search)) {
          options.unshift(this.search)
        }

        return options
      },

      /**
       * Check if there aren't any options selected.
       * @return {Boolean}
       */
      isValueEmpty() {
        if (this.mutableValue) {
          if (typeof this.mutableValue === 'object') {
            return !Object.keys(this.mutableValue).length
          }
          return !this.mutableValue.length
        }

        return true;
      },

      /**
       * Return the current value in array format.
       * @return {Array}
       */
      valueAsArray() {
        if (this.multiple && this.mutableValue && this.mutableValue instanceof Array) {
          return this.mutableValue
        } else if (this.mutableValue) {
          return [this.mutableValue]
        }

        return []
      },

      /**
       * @return {String}
       */
      componentIdAttr() {
        if (this.inputId === '') {
          return ''
        }
        return `${this.inputId}_component`
      },

      /**
       * @return {String}
       */
      toggleIdAttr() {
        if (this.inputId === '') {
          return ''
        }
        return `${this.inputId}_toggle`
      },

      /**
       * @return {String}
       */
      toggleLabelIdAttr() {
        if (this.inputId === '') {
          return ''
        }
        return `${this.inputId}_toggle_label`
      },

      /**
       * @return {String}
       */
      dropdownMenuIdAttr() {
        if (this.inputId === '') {
          return ''
        }
        return `${this.inputId}_dropdown_menu`
      },

      /**
       * @return {String}
       */
      searchIdAttr() {
        if (this.inputId === '') {
          return ''
        }
        return `${this.inputId}_search`
      },

      /**
       * @return {String}
       */
      spinnerIdAttr() {
        if (this.inputId === '') {
          return ''
        }
        return `${this.inputId}_processing_spinner`
      },

      /**
       * @return {String}
       */
      spinnerIdattr() {
        if (this.inputId === '') {
          return ''
        }
        return `${this.inputId}_processing_spinner`
      },

      /**
       * @return {String}
       */
      noOptionsIdAttr() {
        if (this.inputId === '') {
          return ''
        }
        return `${this.inputId}_no_options`
      }
    },

  }
</script>
