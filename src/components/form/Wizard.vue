<template>
    <div
        :id="id ? id : ''"
        class="et-form"
        @keyup.right="focusNextTab"
        @keyup.left="focusPrevTab">
        <div
            class="et-form-header"
            v-if="$slots['title']">
            <slot
                name="et-form-title">
                <h4
                    class="et-title">{{ title }}</h4>
                <p
                    class="et-subtitle">{{ subtitle }}</p>
            </slot>
        </div>

        <div class="et-page-navigation">
            <ul
                class="et-nav"
                role="tablist"
                v-if="tabCount > 1">
                <slot
                    name="step"
                    v-for="(tab, index) in tabs"
                    :tab="tab"
                    :index="index"
                    :navigate-to-tab="navigateToTab">
                    <et-step
                        :tab="tab"
                        @click.native="navigateToTab(index)"
                        @keyup.enter.native="navigateToTab(index)"
                        :index="index"/>
                </slot>
            </ul>
            <div class="et-page-content">
                <slot v-bind="slotProps"/>
            </div>
        </div>

        <div class="et-footer et-clearfix">
            <slot name="footer" v-bind="slotProps">
                <div class="et-footer-left" v-if="displayPrevButton">
                    <slot name="prev" v-bind="slotProps">
                        {{ backButtonText }}
                    </slot>
                </div>

                <div class="et-footer-right">
                    <span
                        @click="nextTab"
                        @keyup.enter="nextTab"
                        role="button"
                        tabindex="0"
                        v-if="isLastStep">
                        {{ finishButtonText }}
                    </span>
                    <span
                        @click="nextTab"
                        @keyup.enter="nextTab"
                        role="button"
                        tabindex="0"
                        v-else>
                        {{ nextButtonText }}
                    </span>
                </div>
            </slot>
        </div>
    </div>
</template>

<script>
    import Step from './Step.vue'
    import {isPromise, findElementAndFocus, getFocusedTabIndex} from '../../utils/helpers'

    export default {
        name: 'EtWizard',
        components: {
            [Step.name]: Step
        },

        props: {
            id: {
                type: String,
                default: 'et_form_' + (new Date()).valueOf()
            },
            title: {
                type: String,
                default: 'ET Form'
            },
            subtitle: {
                type: String,
                default: 'Split your flow in multiple steps'
            },
            nextButtonText: {
                type: String,
                default: 'Next'
            },
            backButtonText: {
                type: String,
                default: 'Back'
            },
            finishButtonText: {
                type: String,
                default: 'Submit'
            },
            hideButtons: {
                type: Boolean,
                default: false
            },
            validateOnBack: Boolean,
            /***
             *
             * Index of the initial tab to display
             */
            startIndex: {
                type: Number,
                default: 0,
                validator: (value) => {
                    return value >= 0
                }
            }
        },

        provide() {
            return {
                addTab: this.addTab,
                removeTab: this.removeTab
            }
        },

        data() {
            return {
                activeTabIndex: 0,
                currentPercentage: 0,
                maxStep: 0,
                loading: false,
                tabs: []
            }
        },

        computed: {
            slotProps() {
                return {
                    nextTab: this.nextTab,
                    prevTab: this.prevTab,
                    activeTabIndex: this.activeTabIndex,
                    isLastStep: this.isLastStep
                }
            },
            tabCount() {
                return this.tabs.length
            },
            isLastStep() {
                return this.activeTabIndex === this.tabCount - 1
            },
            displayPrevButton() {
                return this.activeTabIndex !== 0
            },
            stepPercentage() {
                return 1 / (this.tabCount * 2) * 100
            },
            progress() {
                let percentage = 0
                if (this.activeTabIndex > 0) {
                    let stepsToAdd = 1
                    let stepMultiplier = 2
                    percentage = this.stepPercentage *
                        ((this.activeTabIndex * stepMultiplier) + stepsToAdd)
                } else {
                    percentage = this.stepPercentage
                }
                return percentage
            }
        },
        methods: {
            emitTabChange(prevIndex, nextIndex) {
                this.$emit('on-change', prevIndex, nextIndex)
                this.$emit('update:startIndex', nextIndex)
            },
            addTab(item) {
                const index = this.$slots.default.indexOf(item.$vnode)
                item.tabId = `${item.title.replace(/ /g, '')}${index}`
                this.tabs.splice(index, 0, item)
                // if a step is added before the current one, go to it
                if (index < this.activeTabIndex + 1) {
                    this.maxStep = index
                    this.changeTab(this.activeTabIndex + 1, index)
                }
            },
            removeTab(item) {
                const tabs = this.tabs
                const index = tabs.indexOf(item)
                if (index > -1) {
                    // Go one step back if the current step is removed
                    if (index === this.activeTabIndex) {
                        this.maxStep = this.activeTabIndex - 1
                        this.changeTab(this.activeTabIndex, this.activeTabIndex - 1)
                    }
                    if (index < this.activeTabIndex) {
                        this.maxStep = this.activeTabIndex - 1
                        this.activeTabIndex = this.activeTabIndex - 1
                        this.emitTabChange(this.activeTabIndex + 1, this.activeTabIndex)
                    }
                    tabs.splice(index, 1)
                }
            },
            reset() {
                this.maxStep = 0
                this.tabs.forEach((tab) => {
                    tab.checked = false
                })
                this.navigateToTab(0)
            },
            activateAll() {
                this.maxStep = this.tabs.length - 1
                this.tabs.forEach((tab) => {
                    tab.checked = true
                })
            },
            navigateToTab(index) {
                let validate = index > this.activeTabIndex
                if (index <= this.maxStep) {
                    let cb = () => {
                        if (validate && index - this.activeTabIndex > 1) {
                            // validate all steps recursively until destination index
                            this.changeTab(this.activeTabIndex, this.activeTabIndex + 1)
                            this.beforeTabChange(this.activeTabIndex, cb)
                        } else {
                            this.changeTab(this.activeTabIndex, index)
                            this.afterTabChange(this.activeTabIndex)
                        }
                    }
                    if (validate) {
                        this.beforeTabChange(this.activeTabIndex, cb)
                    } else {
                        this.setValidationError(null)
                        cb()
                    }
                }
                return index <= this.maxStep
            },
            nextTab() {
                let cb = () => {
                    if (this.activeTabIndex < this.tabCount - 1) {
                        this.changeTab(this.activeTabIndex, this.activeTabIndex + 1)
                        this.afterTabChange(this.activeTabIndex)
                    } else {
                        this.$emit('on-complete')
                    }
                }
                this.beforeTabChange(this.activeTabIndex, cb)
            },
            prevTab() {
                let cb = () => {
                    if (this.activeTabIndex > 0) {
                        this.setValidationError(null)
                        this.changeTab(this.activeTabIndex, this.activeTabIndex - 1)
                    }
                }
                if (this.validateOnBack) {
                    this.beforeTabChange(this.activeTabIndex, cb)
                } else {
                    cb()
                }
            },

            focusNextTab() {
                let tabIndex = getFocusedTabIndex(this.tabs)
                if (tabIndex !== -1 && tabIndex < this.tabs.length - 1) {
                    let tabToFocus = this.tabs[tabIndex + 1]
                    if (tabToFocus.checked) {
                        findElementAndFocus(tabToFocus.tabId)
                    }
                }
            },
            focusPrevTab() {
                let tabIndex = getFocusedTabIndex(this.tabs)
                if (tabIndex !== -1 && tabIndex > 0) {
                    let toFocusId = this.tabs[tabIndex - 1].tabId
                    findElementAndFocus(toFocusId)
                }
            },
            setLoading(value) {
                this.loading = value
                this.$emit('on-loading', value)
            },
            setValidationError(error) {
                this.tabs[this.activeTabIndex].validationError = error
                this.$emit('on-error', error)
            },
            validateBeforeChange(promiseFn, callback) {
                this.setValidationError(null)
                // we have a promise
                if (isPromise(promiseFn)) {
                    this.setLoading(true)
                    promiseFn.then((res) => {
                        this.setLoading(false)
                        let validationResult = res === true
                        this.executeBeforeChange(validationResult, callback)
                    }).catch((error) => {
                        this.setLoading(false)
                        this.setValidationError(error)
                    })
                    // we have a simple function
                } else {
                    let validationResult = promiseFn === true
                    this.executeBeforeChange(validationResult, callback)
                }
            },
            executeBeforeChange(validationResult, callback) {
                this.$emit('on-validate', validationResult, this.activeTabIndex)
                if (validationResult) {
                    callback()
                } else {
                    this.tabs[this.activeTabIndex].validationError = 'error'
                }
            },
            beforeTabChange(index, callback) {
                if (this.loading) {
                    return
                }
                let oldTab = this.tabs[index]
                if (oldTab && oldTab.beforeChange !== undefined) {
                    let tabChangeRes = oldTab.beforeChange()
                    this.validateBeforeChange(tabChangeRes, callback)
                } else {
                    callback()
                }
            },
            afterTabChange(index) {
                if (this.loading) {
                    return
                }
                let newTab = this.tabs[index]
                if (newTab && newTab.afterChange !== undefined) {
                    newTab.afterChange()
                }
            },
            changeTab(oldIndex, newIndex, emitChangeEvent = true) {
                let oldTab = this.tabs[oldIndex]
                let newTab = this.tabs[newIndex]
                if (oldTab) {
                    oldTab.active = false
                }
                if (newTab) {
                    newTab.active = true
                }
                if (emitChangeEvent && this.activeTabIndex !== newIndex) {
                    this.emitTabChange(oldIndex, newIndex)
                }
                this.activeTabIndex = newIndex
                this.activateTabAndCheckStep(this.activeTabIndex)
                return true
            },
            deactivateTabs() {
                this.tabs.forEach((tab) => {
                    tab.active = false
                })
            },
            activateTab(index) {
                this.deactivateTabs()
                let tab = this.tabs[index]
                if (tab) {
                    tab.active = true
                    tab.checked = true
                }
            },
            activateTabAndCheckStep(index) {
                this.activateTab(index)
                if (index > this.maxStep) {
                    this.maxStep = index
                }
                this.activeTabIndex = index
            },
            initializeTabs() {
                if (this.tabs.length > 0 && this.startIndex === 0) {
                    this.activateTab(this.activeTabIndex)
                }
                if (this.startIndex < this.tabs.length) {
                    this.activateTabAndCheckStep(this.startIndex)
                } else {
                    window.console.warn(`Prop startIndex set to ${this.startIndex} is greater than the number of tabs - ${this.tabs.length}. Make sure that the starting index is less than the number of tabs registered`)
                }
            }
        },
        mounted() {
            this.initializeTabs()
        }
    }
</script>
