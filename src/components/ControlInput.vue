<template>
    <div
        :class="{
            'kiwi-controlinput--focus': has_focus,
            'kiwi-controlinput--show-send': shouldShowSendButton,
            'kiwi-controlinput--show-tools': shouldShowTools,
            'kiwi-controlinput--show-tools--inline': shouldShowToolsInline,
            'kiwi-controlinput--selfuser-open': selfuser_open,
        }"
        class="kiwi-controlinput kiwi-theme-bg"
    >
        <div class="kiwi-controlinput-selfuser">
            <transition name="kiwi-selfuser-trans">
                <self-user
                    v-if="networkState==='connected'
                        && selfuser_open === true"
                    :network="buffer.getNetwork()"
                    @close="selfuser_open=false"
                />
            </transition>
        </div>

        <div class="kiwi-controlinput-inner">
            <away-status-indicator
                v-if="buffer.getNetwork() && buffer.getNetwork().state === 'connected'"
                :network="buffer.getNetwork()"
                :user="buffer.getNetwork().currentUser()"
            />
            <div v-if="currentNick" class="kiwi-controlinput-user" @click="toggleSelfUser">
                <span class="kiwi-controlinput-user-nick">{{ currentNick }}</span>
                <i
                    :class="[selfuser_open ? 'fa-caret-down' : 'fa-caret-up']"
                    class="fa"
                    aria-hidden="true"
                />
            </div>
            <form
                class="kiwi-controlinput-form"
                @submit.prevent="submitForm"
                @click="maybeHidePlugins"
            >
                <auto-complete
                    v-if="autocomplete_open"
                    ref="autocomplete"
                    :items="autocomplete_items"
                    :filter="autocomplete_filter"
                    :buffer="buffer"
                    @temp="onAutocompleteTemp"
                    @selected="onAutocompleteSelected"
                    @cancel="onAutocompleteCancel"
                />
                <typing-users-list v-if="buffer.setting('share_typing')" :buffer="buffer" />
                <div class="kiwi-controlinput-input-wrap">
                    <irc-input
                        ref="input"
                        :placeholder="$t('input_placeholder')"
                        class="kiwi-controlinput-input"
                        wrap="off"
                        @input="inputUpdate"
                        @keydown="inputKeyDown($event)"
                        @keyup="inputKeyUp($event)"
                        @click="closeToolsPlugins"
                        @focus="focusChanged"
                        @blur="focusChanged"
                    />
                </div>
                <div
                    v-if="shouldShowSendButton"
                    class="kiwi-controlinput-send-container kiwi-controlinput-tools"
                >
                    <button
                        ref="sendButton"
                        type="submit"
                        class="kiwi-controlinput-button kiwi-controlinput-send fa fa-paper-plane"
                    />
                </div>
            </form>

            <div
                v-if="shouldShowTools"
                ref="plugins"
                class="kiwi-controlinput-tools kiwi-controlinput-tools-wrapper"
            >
                <div
                    v-if="!shouldShowToolsInline"
                    class="kiwi-controlinput-tools-expand kiwi-controlinput-button"
                    :class="{'kiwi-controlinput-tools-expand--closed': !showPlugins}"
                    @click="showPlugins=!showPlugins"
                >
                    <i class="fa fa-bars" aria-hidden="true" />
                </div>
                <transition name="kiwi-plugin-ui-trans">
                    <div
                        v-if="showPlugins || shouldShowToolsInline"
                        class="kiwi-controlinput-tools-container"
                    >
                        <div
                            v-if="shouldShowColorPicker"
                            class="kiwi-controlinput-button"
                            @click.prevent="onToolClickTextStyle"
                        >
                            <i class="fa fa-adjust" aria-hidden="true" />
                        </div>
                        <div
                            v-if="shouldShowEmojiPicker"
                            class="kiwi-controlinput-button"
                            @click.prevent="onToolClickEmoji"
                        >
                            <i class="fa fa-smile-o" aria-hidden="true" />
                        </div>
                        <div
                            v-for="plugin in pluginUiElements"
                            :key="plugin.id"
                            v-rawElement="{
                                el: plugin.el,
                                props: {
                                    kiwi: {
                                        buffer: buffer,
                                        controlinput: self,
                                    }
                                }
                            }"
                            class="kiwi-controlinput-button"
                        />
                    </div>
                </transition>
            </div>
        </div>

        <div class="kiwi-controlinput-active-tool">
            <component :is="active_tool" v-bind="active_tool_props" />
        </div>
    </div>
</template>

<script>
'kiwi public';

import _ from 'lodash';
import * as TextFormatting from '@/helpers/TextFormatting';
import * as settingTools from '@/libs/settingTools';
import autocompleteCommands from '@/res/autocompleteCommands';
import GlobalApi from '@/libs/GlobalApi';
import AutoComplete from './AutoComplete';
import ToolTextStyle from './inputtools/TextStyle';
import ToolEmoji from './inputtools/Emoji';
import SelfUser from './SelfUser';
import AwayStatusIndicator from './AwayStatusIndicator';
import TypingUsersList from './TypingUsersList';

export default {
    components: {
        AutoComplete,
        AwayStatusIndicator,
        SelfUser,
        TypingUsersList,
    },
    props: ['container', 'buffer'],
    data() {
        return {
            self: this,
            selfuser_open: false,
            autocomplete_open: false,
            autocomplete_items: [],
            autocomplete_filter: '',
            // Not filtering through the autocomplete list means that the entire word is put
            // in place when cycling through items. Just as with traditional IRC clients when
            // tabbing through nicks.
            // When filtering through the list, we keep typing more of the word we want as the
            // autocomplete list filters its results to show us the relevant items, not replacing
            // the current word until we select an item.
            autocomplete_filtering: true,
            active_tool: null,
            active_tool_props: {},
            pluginUiElements: GlobalApi.singleton().controlInputPlugins,
            showPlugins: false,
            current_input_value: '',
            has_focus: false,
            keep_focus: false,
        };
    },
    computed: {
        currentNick() {
            let activeNetwork = this.$state.getActiveNetwork();
            return activeNetwork ?
                activeNetwork.nick :
                '';
        },
        networkState() {
            let activeNetwork = this.$state.getActiveNetwork();
            return activeNetwork ?
                activeNetwork.state :
                '';
        },
        shouldShowSendButton() {
            return this.$state.ui.is_touch || this.$state.setting('showSendButton');
        },
        shouldShowEmojiPicker() {
            return this.$state.setting('showEmojiPicker') && !this.$state.ui.is_touch;
        },
        shouldShowColorPicker() {
            return this.$state.setting('showColorPicker');
        },
        shouldShowTools() {
            if (
                this.pluginUiElements.length ||
                this.shouldShowEmojiPicker ||
                this.shouldShowColorPicker
            ) {
                return true;
            }
            return false;
        },
        shouldShowToolsInline() {
            let toolCount = this.pluginUiElements.length;
            if (this.shouldShowEmojiPicker) {
                toolCount++;
            }
            if (this.shouldShowColorPicker) {
                toolCount++;
            }

            if (toolCount === 1) {
                // No point showing a menu button to replace one item
                return true;
            }

            // Button size (36px)
            // Total buttons width < 1/5 screen width
            return (36 * toolCount < this.$state.ui.app_width / 5);
        },
        history() {
            if (this.$state.setting('buffers.shared_input')) {
                return this.$state.ui.input_history;
            }
            return this.buffer.input_history;
        },
        history_pos: {
            get() {
                if (this.$state.setting('buffers.shared_input')) {
                    return this.$state.ui.input_history_pos;
                }
                return this.buffer.input_history_pos;
            },
            set(newVal) {
                if (this.$state.setting('buffers.shared_input')) {
                    this.$state.ui.input_history_pos = newVal;
                } else {
                    this.buffer.input_history_pos = newVal;
                }
            },
        },
    },
    watch: {
        history_pos(newVal) {
            let val = this.history[this.history_pos];
            this.$refs.input.setValue(val || '');
        },
        buffer() {
            if (!this.$state.setting('buffers.shared_input')) {
                this.inputRestore();
            }

            this.autocomplete_open = false;
        },
    },
    created() {
        this.typingTimer = null;
        this.lastTypingTime = 0;

        this.listen(this.$state, 'document.keydown', (ev) => {
            // No input box currently? Nothing to shift focus to
            if (!this.$refs.input) {
                return;
            }

            // If we're copying text, don't shift focus
            if (ev.ctrlKey || ev.altKey || ev.metaKey) {
                return;
            }

            // shift key on its own, don't shift focus we handle this below
            if (ev.keyCode === 16) {
                return;
            }

            // Firefox 66.0.3 on linux isn't consistently setting ev.ctrlKey === true when only
            // the control key is pressed so add a specific check for this
            // TODO: Remove this check once ff 66.0.3 is no longer around
            if (ev.keyCode === 17) {
                return;
            }

            // If we are using shift and arrow keys, don't shift focus
            // this allows users to adjust text selection
            let arrowKeyCodes = [37, 38, 39, 40];
            if (ev.shiftKey && arrowKeyCodes.indexOf(ev.keyCode) !== -1) {
                return;
            }

            // If we're typing into an input box somewhere, ignore
            let elements = ['input', 'select', 'textarea', 'button', 'datalist', 'keygen'];
            let doNotRefocus =
                elements.indexOf(ev.target.tagName.toLowerCase()) > -1 ||
                ev.target.getAttribute('contenteditable');

            if (doNotRefocus) {
                return;
            }

            this.$refs.input.focus();
        });

        this.listen(this.$state, 'input.insertnick', (nick) => {
            if (!this.$refs.input) {
                return;
            }

            let val = nick;
            if (this.current_input_value === '') {
                val += ': ';
            } else {
                val += ' ';
            }

            this.$refs.input.insertText(val);
        });

        this.listen(this.$state, 'input.tool', (toolComponent) => {
            this.toggleInputTool(toolComponent);
        });
    },
    mounted() {
        this.inputRestore();
    },
    methods: {
        inputUpdate(val) {
            this.current_input_value = val;

            if (this.$state.setting('buffers.shared_input')) {
                this.$state.ui.current_input = val;
            } else {
                this.buffer.current_input = val;
            }

            this.maybeHidePlugins();
        },
        inputRestore() {
            let currentInput = this.$state.setting('buffers.shared_input') ?
                this.$state.ui.current_input :
                this.buffer.current_input;

            this.$refs.input.reset(currentInput, this.keep_focus);
            this.$refs.input.selectionToEnd();
        },
        toggleSelfUser() {
            if (this.networkState === 'connected') {
                this.selfuser_open = !this.selfuser_open;
            }
        },
        maybeHidePlugins() {
            // Save some space if we're typing on a small screen
            if (this.$state.ui.app_width < 500) {
                this.showPlugins = false;
            }
        },
        onToolClickTextStyle() {
            this.toggleInputTool(ToolTextStyle);
        },
        onToolClickEmoji() {
            this.toggleInputTool(ToolEmoji);
        },
        closeToolsPlugins() {
            this.showPlugins = false;
            this.closeInputTool();
        },
        closeInputTool() {
            this.active_tool = null;
        },
        toggleInputTool(tool) {
            if (!tool || this.active_tool === tool) {
                this.active_tool = null;
            } else {
                this.active_tool_props = {
                    buffer: this.buffer,
                    ircinput: this.$refs.input,
                };
                this.active_tool = tool;
            }
        },
        toggleBold() {
            this.$refs.input.toggleBold();
        },
        toggleItalic() {
            this.$refs.input.toggleItalic();
        },
        toggleUnderline() {
            this.$refs.input.toggleUnderline();
        },
        onAutocompleteCancel() {
            this.autocomplete_open = false;
        },
        onAutocompleteTemp(selectedValue, selectedItem) {
            if (!this.autocomplete_filtering) {
                this.$refs.input.setCurrentWord(selectedValue);
            }
        },
        onAutocompleteSelected(selectedValue, selectedItem) {
            let word = selectedValue;
            if (word.length > 0) {
                this.$refs.input.setCurrentWord(word);
            }
            this.autocomplete_open = false;
        },
        inputKeyDown(event) {
            let meta = false;

            if (navigator.appVersion.indexOf('Mac') !== -1) {
                meta = event.metaKey;
            } else {
                meta = event.ctrlKey;
            }

            // If autocomplete has handled the event, don't also handle it here
            if (this.autocomplete_open && this.$refs.autocomplete.handleOnKeyDown(event)) {
                return;
            }

            // When not filtering, select the current autocomplete item so that we can type any
            // character directly after a nick
            if (this.autocomplete_open && !this.autocomplete_filtering) {
                this.$refs.autocomplete.selectCurrentItem();
            }

            if (event.keyCode === 13 && (
                (event.altKey && !event.shiftKey && !event.metaKey && !event.ctrlKey) ||
                (event.shiftKey && !event.altKey && !event.metaKey && !event.ctrlKey)
            )) {
                // Add new line when shift+enter or alt+enter is pressed
                event.preventDefault();
                this.$refs.input.insertText('\n');
            } else if (event.keyCode === 13) {
                // Send message when enter is pressed
                event.preventDefault();
                this.submitForm();
            } else if (event.keyCode === 32) {
                // Hitting space after just typing an ascii emoji will get it replaced with
                // its image
                if (this.$state.setting('buffers.show_emoticons')) {
                    let currentWord = this.$refs.input.getCurrentWord();
                    let emojiList = this.$state.setting('emojis');
                    if (emojiList.hasOwnProperty(currentWord.word)) {
                        let emoji = emojiList[currentWord.word];
                        let url = this.$state.setting('emojiLocation') + emoji;
                        this.$refs.input.setCurrentWord('');
                        this.$refs.input.addImg(currentWord.word + ' ', url);
                    }
                }
            } else if (event.keyCode === 38) {
                // Up
                if (this.$refs.input.getCaretIdx() > 0) {
                    // not at the start of input, allow normal input behaviour
                    return;
                }

                event.preventDefault();
                this.historyBack();
            } else if (event.keyCode === 40) {
                // Down
                let end = this.$refs.input.getRawText().replace(/\r?\n/g, '').length;
                if (this.$refs.input.getCaretIdx() < end) {
                    // not at the end of input, allow normal input behaviour
                    return;
                }
                event.preventDefault();
                this.historyForward();
                this.$nextTick(() => {
                    this.$refs.input.selectionToEnd();
                });
            } else if (
                event.keyCode === 9
                && !event.shiftKey
                && !event.altKey
                && !event.metaKey
                && !event.ctrlKey
            ) {
                // Tab and no other keys as tab+other is often a keyboard shortcut
                // Tab key was just pressed, start general auto completion
                let currentWord = this.$refs.input.getCurrentWord();
                let currentToken = currentWord.word.substr(0, currentWord.position);
                let inputText = this.$refs.input.getRawText();

                let items = [];
                if (inputText.indexOf('/set') === 0) {
                    items = this.buildAutoCompleteItems({ settings: true });
                } else {
                    items = this.buildAutoCompleteItems({
                        users: true,
                        buffers: true,
                    });
                }

                this.openAutoComplete(items);
                this.autocomplete_filter = currentToken;

                // Disable filtering so that tabbing cycles through words more like
                // traditional IRC clients.
                this.autocomplete_filtering = false;
                event.preventDefault();
            } else if (meta && event.keyCode === 75) {
                // meta + k
                this.toggleInputTool(ToolTextStyle);
                event.preventDefault();
            } else if (meta && event.keyCode === 66) {
                // meta + b
                this.toggleBold();
                event.preventDefault();
            } else if (meta && event.keyCode === 73) {
                // meta + i
                this.toggleItalic();
                event.preventDefault();
            } else if (meta && event.keyCode === 85) {
                // meta + u
                this.toggleUnderline();
                event.preventDefault();
            }
        },
        inputKeyUp(event) {
            let inputVal = this.$refs.input.getRawText();
            let currentWord = this.$refs.input.getCurrentWord();
            let currentToken = currentWord.word.substr(0, currentWord.position);
            let autocompleteTokens = this.$state.setting('autocompleteTokens');

            if (event.keyCode === 27 && this.autocomplete_open) {
                this.autocomplete_open = false;
            } else if (this.autocomplete_open && currentToken === '') {
                this.autocomplete_open = false;
            } else if (this.autocomplete_open) {
                // @ is a shortcut to open the nicklist autocomplete. It's not part
                // of the nick so strip it out before passing currentToken to the
                // filter.
                if (currentToken[0] === '@') {
                    currentToken = currentToken.substr(1);
                }
            } else if (currentToken === '@' && autocompleteTokens.includes('@')) {
                // Just typed @ so start the nick auto completion
                this.openAutoComplete(this.buildAutoCompleteItems({ users: true }));
                this.autocomplete_filtering = true;
            } else if (inputVal === '/' && autocompleteTokens.includes('/')) {
                // Just typed / so start the command auto completion
                this.openAutoComplete(this.buildAutoCompleteItems({ commands: true }));
                this.autocomplete_filtering = true;
            } else if (currentToken === '#' && autocompleteTokens.includes('#')) {
                // Just typed # so start the command auto completion
                this.openAutoComplete(this.buildAutoCompleteItems({ buffers: true }));
                this.autocomplete_filtering = true;
            } else if (
                event.keyCode === 9
                && !event.shiftKey
                && !event.altKey
                && !event.metaKey
                && !event.ctrlKey
            ) {
                // Tab and no other keys as tab+other is often a keyboard shortcut
                event.preventDefault();
            } else if (!event.key.match(/^(Shift|Control|Alt|Enter)/)) {
                if (inputVal[0] === '/') {
                    // Don't send typing status for commands
                    return;
                }

                if (inputVal.trim()) {
                    this.startTyping();
                } else {
                    this.stopTyping(true);
                }
            }

            if (this.autocomplete_open && this.autocomplete_filtering) {
                this.autocomplete_filter = currentToken;
            }
        },
        submitForm() {
            let rawInput = this.$refs.input.getValue();
            if (!rawInput) {
                if (!this.has_focus && this.keep_focus) {
                    // Maybe triggered by the send button on empty input,
                    // put focus back into the input
                    this.$refs.input.focus();
                }
                return;
            }

            let ircText = this.$refs.input.buildIrcText();
            this.$state.$emit('input.raw', ircText);

            this.historyAdd(rawInput);

            this.$refs.input.reset('', this.keep_focus);

            this.stopTyping(false);
        },
        historyAdd(rawInput) {
            // Add to history, keeping the history trimmed to the last 50 entries
            this.history.push(rawInput);
            this.history.splice(0, this.history.length - 50);
            this.history_pos = this.history.length;
        },
        historyBack() {
            let rawText = this.$refs.input.getRawText();
            let rawInput = this.$refs.input.getValue();
            if (rawText.trim() && this.history_pos === this.history.length) {
                this.historyAdd(rawInput);
                this.history_pos--;
            }
            if (this.history_pos > 0) {
                this.history_pos--;
            }
        },
        historyForward() {
            // Purposely let history_pos go 1 index beyond the history length
            // so that we can detect if we're not currently using a history value
            if (this.history_pos < this.history.length) {
                this.history_pos++;
            }
        },
        focusChanged(event) {
            this.has_focus = event.type === 'focus';

            // iOS does not populate relatedTarget when the relatedTarget is the send button
            // leaving it impossible to detect if returning focus is required
            if (
                event.type === 'blur' &&
                event.relatedTarget &&
                event.relatedTarget === this.$refs.sendButton
            ) {
                // new target is the send button, keep focus on reset
                return;
            }
            this.keep_focus = event.type === 'focus';
        },
        openAutoComplete(items) {
            if (this.$state.setting('showAutocomplete')) {
                this.autocomplete_items = items;
                this.autocomplete_open = true;
            }
        },
        buildAutoCompleteItems(_opts) {
            let opts = _opts || {};
            let list = [];

            if (opts.users) {
                let userList = _.values(this.buffer.users).map((user) => {
                    let item = {
                        text: user.nick,
                        type: 'user',
                    };
                    return item;
                });

                if (this.buffer.isQuery()) {
                    userList.push({
                        text: this.buffer.name,
                        type: 'user',
                    });
                }

                list = list.concat(userList);
            }

            if (opts.buffers) {
                let bufferList = [];
                this.buffer.getNetwork().buffers.forEach((buffer) => {
                    if (buffer.isChannel()) {
                        bufferList.push({
                            text: buffer.name,
                            type: 'buffer',
                        });
                    }
                });

                list = list.concat(bufferList);
            }

            if (opts.commands) {
                let commandList = [];
                autocompleteCommands.forEach((command) => {
                    // allow descriptions to be translation keys or static strings
                    let desc = command.description.indexOf('locale_id_') === 0 ?
                        TextFormatting.t(command.description.substr(10)) :
                        command.description;
                    commandList.push({
                        text: '/' + command.command,
                        description: desc,
                        type: 'command',
                        // Each alias needs the / command prefix adding
                        alias: (command.alias || []).map((c) => '/' + c),
                    });
                });

                list = list.concat(commandList);
            }

            if (opts.settings) {
                let out = {};
                let base = [];
                settingTools.buildTree(out, base, this.$state.getSetting('settings'), false);
                settingTools.buildTree(out, base, this.$state.getSetting('user_settings'), true);

                let settingList = [];
                Object.keys(out).forEach((setting) => {
                    settingList.push({
                        text: setting,
                        type: 'setting',
                    });
                });

                list = list.concat(settingList);
            }

            return list;
        },
        startTyping() {
            let network = this.buffer.getNetwork();
            if (!network.ircClient.network.cap.isEnabled('message-tags')) {
                return;
            }
            if (!this.buffer || !this.buffer.shouldShareTyping()) {
                return;
            }

            if (this.typingTimer) {
                clearTimeout(this.typingTimer);
                this.typingTimer = null;
            }
            this.typingTimer = setTimeout(() => this.stopTyping(true), 3000);

            if (Date.now() < this.lastTypingTime + 3000) {
                return;
            }

            network.ircClient.typing.start(this.buffer.name);

            this.lastTypingTime = Date.now();
        },
        stopTyping(sendStop) {
            let network = this.buffer.getNetwork();
            if (!network.ircClient.network.cap.isEnabled('message-tags')) {
                return;
            }
            if (!this.buffer || !this.buffer.shouldShareTyping()) {
                return;
            }

            if (this.typingTimer) {
                clearTimeout(this.typingTimer);
                this.typingTimer = null;
                this.lastTypingTime = 0;
            }

            this.$refs.input.getRawText().trim() ?
                network.ircClient.typing.pause(this.buffer.name) :
                network.ircClient.typing.stop(this.buffer.name, sendStop);
        },
    },
};
</script>

<style lang="less">

.kiwi-controlinput {
    z-index: 999;
    position: relative;
    border-top: 1px solid;
}

.kiwi-controlinput,
.kiwi-controlinput-inner {
    padding: 0;
    box-sizing: border-box;
    transition: width 0.2s;
    transition-delay: 0.2s;
}

.kiwi-controlinput-inner {
    display: flex;
    position: relative;
    height: 100%;

    .kiwi-awaystatusindicator {
        margin-top: 16px;
        margin-left: 10px;
        margin-right: -2px;
    }
}

.kiwi-controlinput-user {
    height: 100%;
    padding-left: 10px;
    font-weight: bold;
    text-align: center;
    cursor: pointer;
    line-height: 40px;
    transition: width 0.2s;
    transition-delay: 0.1s;

    > i {
        font-size: 120%;
        margin-left: 8px;
    }
}

.kiwi-controlinput--selfuser-open {
    .kiwi-controlinput-user {
        width: 296px;
        transition: width 0.2s;
        transition-delay: 0.1s;
    }

    .kiwi-controlinput-selfuser {
        width: 324px;
        max-height: 300px;
        opacity: 1;
    }
}

.kiwi-controlinput-form {
    flex: 1;
    overflow: hidden;
    display: flex;
    box-sizing: border-box;
}

.kiwi-controlinput-input {
    text-align: left;
    height: 100%;
    outline: none;
    border: none;
}

.kiwi-controlinput-input-wrap {
    width: 100%;
    height: 100%;
    box-sizing: border-box;
    overflow: hidden;
    margin: 0 10px;
}

.kiwi-controlinput-active-tool {
    position: absolute;
    bottom: calc(100% + 1px);
    right: 74px;
    left: 0;
    z-index: 1;
}

.kiwi-controlinput-selfuser {
    position: absolute;
    bottom: 0;
    z-index: 10;
    left: 0;
    max-height: 0;
    width: 324px;
    box-sizing: border-box;
    border-radius: 0 6px 0 0;
    opacity: 0;
    border-top: 1px solid;
    border-right: 1px solid;
    overflow: hidden;
}

.kiwi-selfuser-trans-enter,
.kiwi-selfuser-trans-leave-to {
    opacity: 0;
    height: 0;
}

.kiwi-selfuser-trans-enter-to,
.kiwi-selfuser-trans-leave {
    opacity: 1;
}

.kiwi-selfuser-trans-enter-active,
.kiwi-selfuser-trans-leave-active {
    transition: all 0.4s;
}

.kiwi-controlinput-tools {
    border-radius: 8px;
    padding: 1px;
    height: 36px;
}

.kiwi-controlinput-tools-expand > i {
    transition: transform 0.2s;
}

.kiwi-controlinput-tools-expand--closed > i {
    transform: rotateZ(180deg);
}

.kiwi-controlinput-send {
    padding: 1px 6px;
}

.kiwi-controlinput--show-send.kiwi-controlinput--show-tools {
    // The send button and tools are visible, merge their borders
    .kiwi-controlinput-tools-wrapper {
        border-radius: 0 8px 8px 0;
        padding: 1px 1px 1px 0;
    }

    .kiwi-controlinput-send-container {
        border-radius: 8px 0 0 8px;
        padding: 1px 0 1px 1px;
    }
}

.kiwi-controlinput-tools-container {
    display: flex;
    flex-wrap: wrap-reverse;
    flex-direction: row-reverse;
    padding: 1px;
    border-radius: 8px;
    position: absolute;
    bottom: calc(100% + 1px);
    top: auto;
    right: 0;
    width: 72px;
}

.kiwi-controlinput--show-tools--inline {
    .kiwi-controlinput-tools-container {
        flex-direction: row;
        position: relative;
        width: auto;
        top: 0;
        padding: 0;
    }

    .kiwi-controlinput-active-tool {
        right: 0;
    }
}

.kiwi-controlinput-button {
    display: inline-block;
    width: 34px;
    height: 34px;
    margin: 1px;
    text-align: center;
    border-radius: 8px;
    box-sizing: border-box;
    cursor: pointer;

    i {
        font-size: 20px;
        line-height: 32px;
        margin: 0;
    }
}

.kiwi-plugin-ui-trans-enter,
.kiwi-plugin-ui-trans-leave-to {
    right: -100%;
}

.kiwi-plugin-ui-trans-enter-to,
.kiwi-plugin-ui-trans-leave {
    right: 0;
}

.kiwi-plugin-ui-trans-enter-active,
.kiwi-plugin-ui-trans-leave-active {
    transition: right 0.2s;
}

@media screen and (max-width: 500px) {
    .kiwi-controlinput-user-nick {
        display: none;
    }

    .kiwi-controlinput-user > i {
        margin-left: 0;
    }
}

@media screen and (max-width: 769px) {
    .kiwi-controlinput--selfuser-open .kiwi-controlinput-selfuser {
        width: 100%;
        border-radius: 0;
        border-right: 0;
    }

    .kiwi-wrap--statebrowser-drawopen .kiwi-controlinput {
        z-index: 0;
    }

    // hide the control input on narrow screens when the self user box is open
    .kiwi-controlinput--selfuser-open .kiwi-controlinput-inner {
        display: none;
    }

    .kiwi-controlinput-tools-container {
        width: 34px;
    }

    .kiwi-controlinput-active-tool {
        right: 36px;
    }
}

.kiwi-typinguserslist {
    position: absolute;
    top: -24px;
    background: var(--brand-default-bg);
}
</style>
