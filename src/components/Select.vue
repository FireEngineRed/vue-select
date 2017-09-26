<style>
  .dropdown-menu li .disabled {
    background-color: #f8f8f8;
    cursor: not-allowed;
  }
  /* Dropdown Menu */
  .v-select .dropdown-menu {
    overflow-y: scroll;
  }
  .v-select .no-options {
    text-align: center;
    margin: 0 20px;
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
  .fade-enter-active,
  .fade-leave-active {
    transition: opacity .15s cubic-bezier(1.0, 0.5, 0.8, 1.0);
  }
  .fade-enter,
  .fade-leave-to {
    opacity: 0;
  }

  /* Processing Spinner */
  .processingSpinner {
      margin: 0 auto;
      width: 70px;
      text-align: center;
  }

  .processingSpinner > div {
      width: 18px;
      height: 18px;
      background-color: #343d41;
      border-radius: 100%;
      display: inline-block;
      -webkit-animation: bouncedelay 1.4s infinite ease-in-out;
      animation: bouncedelay 1.4s infinite ease-in-out;
      /* Prevent first frame from flickering when animation starts */
      -webkit-animation-fill-mode: both;
      animation-fill-mode: both;
  }

  .processingSpinner .bounce1 {
      -webkit-animation-delay: -0.32s;
      animation-delay: -0.32s;
  }

  .processingSpinner .bounce2 {
      -webkit-animation-delay: -0.16s;
      animation-delay: -0.16s;
  }

  @-webkit-keyframes bouncedelay {
      0%, 80%, 100% { -webkit-transform: scale(0.0) }
      40% { -webkit-transform: scale(1.0) }
  }

  @keyframes bouncedelay {
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
  <div class="btn-group v-select" :class="dropdownClasses" :id="componentIdAttr">
    <button
      type="button"
      ref="toggle"
      @keyup.space="toggleDropdown"
      @keyup.enter="toggleDropdown"
      @click.prevent="toggleDropdown"
      @keyup.esc="onEscape"
      @keydown.down.prevent="tabDown"
      @keydown.tab="focusSearch"
      class="btn btn-sm btn-default dropdown-toggle"
      :tabindex="tabindex + 1"
      :id="toggleIdAttr"
    >
      <span :id="toggleLabelIdAttr">{{ dropdownButtonLabel() }}</span>
      <span class="caret"></span>
      <input type="hidden" v-for="(option, index) in valueAsArray" v-bind:key="index" :id="getSelectedOptionIdAttr(index)" :name="getSelectedOptionNameAttr()" :value="getSelectedOptionValue(option)">
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
          <div :id="spinnerIdAttr" class="processingSpinner" v-if="mutableLoading">
            <div class="bounce1"></div>
            <div class="bounce2"></div>
            <div class="bounce3"></div>
          </div>
        </slot>
        <li v-if="!mutableLoading" v-for="(option, index) in valueAsArray" v-bind:key="index" :class="{ active: isOptionSelected(option) }" @mouseover="typeAheadPointer = index">
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
        <li v-if="!mutableLoading" v-for="(option, index) in filteredOptions" v-bind:key="index" :class="{ 'dropdown-header': isOptgroupOption(option) }" @mouseover="typeAheadPointer = index">
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
        <li v-if="!filteredOptions.length && !mutableLoading" :id="noOptionsIdAttr" class="no-options">
          <slot name="no-options">Sorry, no matching options.</slot>
        </li>
      </ul>
    </transition>
  </div>
</template>

<script type="text/babel">
  import pointerScroll from '../mixins/pointerScroll'
  import typeAheadPointer from '../mixins/typeAheadPointer'
  import ajax from '../mixins/ajax'
  import removeDiacritics from '../mixins/removeDiacritics'

  export default {
    mixins: [pointerScroll, typeAheadPointer, ajax, removeDiacritics],

    directives: {
      // v-focus
      focus: {
        inserted: function (el) {
          el.focus()
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
        default: 'fade'
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
      }
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
      }
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
          var index = this.mutableValue.indexOf(ref)
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
          this.open = !this.open
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
          this.open = false
          if (this.clearSearchOnSelect) {
            this.search = ''
          }
          this.$refs.toggle.focus()
        } else {
          this.open = true
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
        this.open = false
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
            this.open = true
            return;
          }
        }

        const tabIndex = parseInt(document.activeElement.getAttribute('tabindex'))
        const nodes = this.$el.querySelectorAll('.v-select-option')
        const lastTabIndex = parseInt(nodes[nodes.length - 1].getAttribute('tabindex'))
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
        const tabIndex = parseInt(document.activeElement.getAttribute('tabindex'))
        const nodes = this.$el.querySelectorAll('.v-select-option')
        const firstTabIndex = parseInt(nodes[0].getAttribute('tabindex'))
        if (tabIndex > firstTabIndex && document.activeElement.parentNode.previousSibling.classList) {
          const prevIndex = document.activeElement.parentNode.previousSibling.classList.contains('dropdown-header') ? 2 : 1
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
        if (typeof option === 'object' && option.hasOwnProperty('value')) {
          return option['value']
        }
        return option
      },

      /**
       * Check if an option is readonly
       */
      isReadonlyOption(option) {
        let opt = JSON.parse(JSON.stringify(option))
        let matches = this.readonlyValues.filter((val) => {
          let optionProps = Object.getOwnPropertyNames(opt)
          let readonlyProps = Object.getOwnPropertyNames(val)

          // If the property lengths don't match,
          // not a match
          if (optionProps.length !== readonlyProps.length) {
            return false
          }

          // If the object property isn't found in the readonly
          // props, they are different, filter it
          for (let i = 0; i < optionProps.length; i += 1) {
            if (!readonlyProps.includes(optionProps[i])) {
              return false
            }
            if (opt[optionProps[i]] !== val[optionProps[i]]) {
              return false
            }
          }

          return true
        })

        return matches.length ? true : false
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
          loading: this.mutableLoading
        }
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
        this.mutableOptions.forEach((option) => {
          if (typeof option === 'object' && option.hasOwnProperty(this.label) && typeof option.value === 'object') {
            this.$set(option, 'optgroup', true)
            flattenedOptions.push(option)
            option.value.forEach((opt) => {
              if (typeof opt === 'object' && opt.hasOwnProperty(this.label)) {
                flattenedOptions.push(opt)
              } else if (typeof option === 'object' && !option.hasOwnProperty(this.label)) {
                return console.warn(`[vue-select warn]: Label key "option.${this.label}" does not exist in options object.\nhttp://sagalbot.github.io/vue-select/#ex-labels`)
              } else {
                flattenedOptions.push(opt)
              }
            })
          } else if (typeof option === 'object' && option.hasOwnProperty(this.label)) {
            unsortedOptions.push(option)
          } else if (typeof option === 'object' && !option.hasOwnProperty(this.label)) {
            return console.warn(`[vue-select warn]: Label key "option.${this.label}" does not exist in options object.\nhttp://sagalbot.github.io/vue-select/#ex-labels`)
          } else {
            unsortedOptions.push(option)
          }
        })
        const sortedOptions = unsortedOptions.concat(flattenedOptions)

        let options = sortedOptions.filter((option) => {
          // Filter out those already selected, since we are looping them differently
          if (this.valueAsArray && this.valueAsArray.includes(option)) {
            return false
          }

          if (typeof option === 'object' && option.hasOwnProperty(this.label) && typeof option.value === 'object') {
            // Optgroups - filter the optgroup values
            let optgroupOptions = option.value.filter((opt) => {
              if (typeof opt === 'object' && opt.hasOwnProperty(this.label)) {
                if (this.removeDiacritics) {
                  return this.removeDiacriticsFromString(opt[this.label]).toLowerCase().indexOf(this.search.toLowerCase()) > -1
                }
                return opt[this.label].toLowerCase().indexOf(this.search.toLowerCase()) > -1
              }
              if (this.removeDiacritics) {
                return this.removeDiacriticsFromString(opt).toLowerCase().indexOf(this.search.toLowerCase()) > -1
              }
              return opt.toLowerCase().indexOf(this.search.toLowerCase()) > -1
            })
            return optgroupOptions.length ? true : false
          } else if (typeof option === 'object' && option.hasOwnProperty(this.label)) {
            if (this.removeDiacritics) {
              return this.removeDiacriticsFromString(option[this.label]).toLowerCase().indexOf(this.search.toLowerCase()) > -1
            }
            return option[this.label].toLowerCase().indexOf(this.search.toLowerCase()) > -1
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
