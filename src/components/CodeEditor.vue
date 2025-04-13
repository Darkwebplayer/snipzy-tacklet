<script setup lang="ts">
import { ref, onMounted, watch, computed } from "vue";
import hljs from "highlight.js";
import "highlight.js/styles/github.css";
import "highlight.js/styles/github-dark.css";

const props = defineProps<{
    modelValue: string;
    darkMode?: boolean;
    wordWrap?: boolean;
    showLineNumbers?: boolean;
    language?: string;
    isHidden?: boolean;
}>();

const emit = defineEmits<{
    "update:modelValue": [value: string];
    "update:darkMode": [value: boolean];
    "update:wordWrap": [value: boolean];
    "update:showLineNumbers": [value: boolean];
    "update:language": [value: string];
    "update:isHidden": [value: boolean];
}>();

// Computed properties for settings with getters and setters
const localDarkMode = computed({
    get: () => props.darkMode,
    set: (value) => emit("update:darkMode", value),
});

const localWordWrap = computed({
    get: () => props.wordWrap,
    set: (value) => emit("update:wordWrap", value),
});

const localShowLineNumbers = computed({
    get: () => props.showLineNumbers ?? true,
    set: (value) => emit("update:showLineNumbers", value),
});

const localLanguage = computed({
    get: () => props.language || "javascript",
    set: (value) => emit("update:language", value),
});

const localIsHidden = computed({
    get: () => props.isHidden ?? false,
    set: (value) => emit("update:isHidden", value),
});

const code = ref(props.modelValue);
const editorRef = ref<HTMLPreElement | null>(null);
const textareaRef = ref<HTMLTextAreaElement | null>(null);
const popoverVisible = ref(false);

// Scroll position tracking
const top = ref(0);
const left = ref(0);

const languages = [
    "javascript",
    "typescript",
    "html",
    "css",
    "python",
    "java",
    "cpp",
    "ruby",
    "php",
    "sql",
];

// Cursor tracking reactive variables
const cursorLine = ref(1);
const cursorColumn = ref(1);

watch(
    () => props.modelValue,
    (newValue) => {
        if (code.value !== newValue) {
            code.value = newValue;
            highlightCode();
        }
    },
);

// Update textarea content and sync with displayed code
const updateContent = () => {
    if (textareaRef.value) {
        code.value = textareaRef.value.value;
        emit("update:modelValue", code.value);
        highlightCode();
        updateCursorPosition();
    }
};

const highlightCode = () => {
    if (editorRef.value) {
        const highlighted = hljs.highlight(code.value, {
            language: localLanguage.value,
        }).value;

        // Replace linebreaks with <br> for proper display
        editorRef.value.innerHTML = highlighted.replace(/\n/g, "<br>");
    }
};

// Handle textarea scroll to keep highlighted code in sync
const handleScroll = (event: Event) => {
    const target = event.target as HTMLTextAreaElement;
    top.value = -target.scrollTop;
    left.value = -target.scrollLeft;
};

// Calculate and update the current cursor position
const updateCursorPosition = () => {
    if (textareaRef.value) {
        const textarea = textareaRef.value;
        const value = textarea.value;
        const selectionStart = textarea.selectionStart;

        // Get text before cursor
        const textBeforeCursor = value.substring(0, selectionStart);
        const lines = textBeforeCursor.split("\n");

        cursorLine.value = lines.length;
        cursorColumn.value = lines[lines.length - 1].length + 1;
    }
};

const copyCode = async () => {
    try {
        await navigator.clipboard.writeText(code.value);

        // Show temporary feedback
        const copyFeedback = document.createElement("div");
        copyFeedback.textContent = "Copied!";
        copyFeedback.className = "copy-feedback";
        document.body.appendChild(copyFeedback);

        setTimeout(() => {
            document.body.removeChild(copyFeedback);
        }, 1500);
    } catch (err) {
        console.error("Failed to copy code:", err);
    }

    // Hide popover after copying
    popoverVisible.value = false;
};

const getLineNumbers = () => {
    return Array.from(
        { length: code.value.split("\n").length },
        (_, i) => i + 1,
    );
};

// Handle tab key in textarea
const handleKeyDown = (event: KeyboardEvent) => {
    if (event.key === "Tab") {
        event.preventDefault();

        const textarea = textareaRef.value!;
        const start = textarea.selectionStart;
        const end = textarea.selectionEnd;

        // Insert tab at cursor position
        const newValue =
            textarea.value.substring(0, start) +
            "  " +
            textarea.value.substring(end);

        // Update the textarea value
        textarea.value = newValue;

        // Move cursor after the inserted tab
        textarea.selectionStart = textarea.selectionEnd = start + 2;

        // Update displayed code
        updateContent();
    }
};

// Toggle popover visibility
const togglePopover = () => {
    popoverVisible.value = !popoverVisible.value;
};

// Click outside handler to close popover
const handleClickOutside = (event: MouseEvent) => {
    const popoverEl = document.querySelector(".popover-menu");
    const settingsButton = document.querySelector(".settings-button");

    if (
        popoverEl &&
        settingsButton &&
        !popoverEl.contains(event.target as Node) &&
        !settingsButton.contains(event.target as Node)
    ) {
        popoverVisible.value = false;
    }
};

watch(
    () => localLanguage.value,
    () => {
        highlightCode();
    },
);

onMounted(() => {
    highlightCode();
    updateCursorPosition();
    document.addEventListener("click", handleClickOutside);
});
</script>

<template>
    <div
        class="code-editor"
        style="min-height: 1200px"
        :class="{
            'dark-mode': localDarkMode,
            'hidden-ui': localIsHidden,
        }"
    >
        <div class="header-bar">
            <div class="language-selector">
                <select v-model="localLanguage">
                    <option v-for="lang in languages" :key="lang" :value="lang">
                        {{ lang }}
                    </option>
                </select>
            </div>

            <div class="header-controls">
                <div class="cursor-position" v-if="!localIsHidden">
                    Line: {{ cursorLine }}, Col: {{ cursorColumn }}
                </div>

                <button
                    class="action-button copy-button"
                    @click="copyCode"
                    title="Copy code"
                    v-if="!localIsHidden"
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="16"
                        height="16"
                        viewBox="0 0 24 24"
                    >
                        <path
                            fill="currentColor"
                            d="M7 6V3a1 1 0 0 1 1-1h12a1 1 0 0 1 1 1v14a1 1 0 0 1-1 1h-3v3c0 .552-.45 1-1.007 1H4.007A1 1 0 0 1 3 21l.003-14c0-.552.45-1 1.006-1zm2 0h8v10h2V4H9z"
                        />
                    </svg>
                </button>

                <button
                    class="action-button settings-button"
                    @click="togglePopover"
                    title="Settings"
                    v-if="!localIsHidden"
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="16"
                        height="16"
                        viewBox="0 0 24 24"
                    >
                        <path
                            fill="currentColor"
                            d="M12 15.5A3.5 3.5 0 0 1 8.5 12 3.5 3.5 0 0 1 12 8.5a3.5 3.5 0 0 1 3.5 3.5 3.5 3.5 0 0 1-3.5 3.5zm7.43-2.53c.04-.32.07-.64.07-.97 0-.33-.03-.66-.07-1l2.11-1.63c.19-.15.24-.42.12-.64l-2-3.46c-.12-.22-.39-.31-.61-.22l-2.49 1c-.52-.39-1.06-.73-1.69-.98l-.37-2.65A.506.506 0 0 0 14 2h-4c-.25 0-.46.18-.5.42l-.37 2.65c-.63.25-1.17.59-1.69.98l-2.49-1c-.22-.09-.49 0-.61.22l-2 3.46c-.13.22-.07.49.12.64L4.57 11c-.04.34-.07.67-.07 1 0 .33.03.65.07.97l-2.11 1.66c-.19.15-.25.42-.12.64l2 3.46c.12.22.39.3.61.22l2.49-1.01c.52.4 1.06.74 1.69.99l.37 2.65c.04.24.25.42.5.42h4c.25 0 .46-.18.5-.42l.37-2.65c.63-.26 1.17-.59 1.69-.99l2.49 1.01c.22.08.49 0 .61-.22l2-3.46c.12-.22.07-.49-.12-.64l-2.11-1.66z"
                        />
                    </svg>
                </button>

                <!-- Popover Menu -->
                <div
                    class="popover-menu"
                    v-if="popoverVisible && !localIsHidden"
                >
                    <div class="popover-item">
                        <label>
                            <input type="checkbox" v-model="localDarkMode" />
                            Dark Mode
                        </label>
                    </div>
                    <div class="popover-item">
                        <label>
                            <input type="checkbox" v-model="localWordWrap" />
                            Wrap Text
                        </label>
                    </div>
                    <div class="popover-item">
                        <label>
                            <input
                                type="checkbox"
                                v-model="localShowLineNumbers"
                            />
                            Line Numbers
                        </label>
                    </div>
                </div>
            </div>
        </div>

        <div
            class="editor-container"
            :class="{
                'with-line-numbers': localShowLineNumbers && !localIsHidden,
            }"
        >
            <div
                v-if="localShowLineNumbers && !localIsHidden"
                class="line-numbers"
            >
                <div
                    v-for="num in getLineNumbers()"
                    :key="num"
                    class="line-number"
                >
                    {{ num }}
                </div>
            </div>

            <div class="code-area">
                <!-- Textarea for actual editing -->
                <textarea
                    ref="textareaRef"
                    v-model="code"
                    @input="updateContent"
                    @scroll="handleScroll"
                    @keydown="handleKeyDown"
                    @keyup="updateCursorPosition"
                    @click="updateCursorPosition"
                    spellcheck="false"
                    :class="{ 'wrap-text': localWordWrap }"
                ></textarea>

                <!-- Pre element for syntax highlighting -->
                <pre
                    ref="editorRef"
                    :class="[
                        `language-${localLanguage}`,
                        { 'wrap-text': localWordWrap },
                    ]"
                    :style="{ top: top + 'px', left: left + 'px' }"
                ></pre>
            </div>
        </div>
    </div>
</template>

<style>
/* Global styles for copy feedback */
.copy-feedback {
    position: fixed;
    top: 20px;
    right: 20px;
    background-color: #4caf50;
    color: white;
    padding: 8px 16px;
    border-radius: 4px;
    z-index: 9999;
    animation: fadeOut 1.5s forwards;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.2);
}

@keyframes fadeOut {
    0% {
        opacity: 1;
    }
    70% {
        opacity: 1;
    }
    100% {
        opacity: 0;
    }
}
</style>

<style scoped>
.code-editor {
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    overflow: hidden;
    background: #fff;
    color: #333;
    font-family: "Menlo", "Monaco", "Consolas", monospace;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
    display: flex;
    flex-direction: column;
}

.dark-mode {
    background: #1e1e1e;
    color: #fff;
    border-color: #444;
}

.header-bar {
    padding: 8px 12px;
    background: #f6f8fa;
    border-bottom: 1px solid #e0e0e0;
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.dark-mode .header-bar {
    background: #2d2d2d;
    border-bottom-color: #444;
}

.language-selector select {
    padding: 4px 8px;
    border: 1px solid #e0e0e0;
    border-radius: 4px;
    background: white;
    color: #333;
    font-size: 13px;
    outline: none;
}

.dark-mode .language-selector select {
    background: #3a3a3a;
    color: #eee;
    border-color: #555;
}

.header-controls {
    display: flex;
    align-items: center;
    gap: 8px;
    position: relative;
}

.cursor-position {
    font-family: "Menlo", "Monaco", "Consolas", monospace;
    font-size: 12px;
    color: #6e7781;
    padding: 4px 8px;
    background: rgba(0, 0, 0, 0.04);
    border-radius: 4px;
}

.dark-mode .cursor-position {
    color: #aaa;
    background: rgba(255, 255, 255, 0.08);
}

.action-button {
    background: transparent;
    border: none;
    cursor: pointer;
    border-radius: 4px;
    padding: 6px;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #6e7781;
    transition: background-color 0.2s;
}

.action-button:hover {
    background-color: rgba(0, 0, 0, 0.05);
    color: #333;
}

.dark-mode .action-button {
    color: #aaa;
}

.dark-mode .action-button:hover {
    background-color: rgba(255, 255, 255, 0.1);
    color: #fff;
}

.editor-container {
    display: flex;
    position: relative;
    flex-grow: 1;
}

.line-numbers {
    padding: 12px 8px;
    background: #f8f9fa;
    border-right: 1px solid #e0e0e0;
    color: #8a919a;
    user-select: none;
    text-align: right;
    min-width: 34px;
    z-index: 5;
}

.dark-mode .line-numbers {
    background: #252525;
    border-right-color: #444;
    color: #6e7781;
}

.line-number {
    font-family: "Menlo", "Monaco", "Consolas", monospace;
    font-size: 13px;
    line-height: 1.5;
    padding: 0 4px;
}

.code-area {
    position: relative;
    flex-grow: 1;
    overflow: hidden;
}

textarea {
    font-family: "Menlo", "Monaco", "Consolas", monospace;
    font-size: 13px;
    line-height: 1.5;
    padding: 12px;
    width: 100%;
    height: 100%;
    min-height: 240px;
    border: none;
    outline: none;
    resize: none;
    background: transparent;
    color: transparent;
    caret-color: #333;
    white-space: pre;
    overflow: auto;
    position: absolute;
    top: 0;
    left: 0;
    z-index: 10;
}

.dark-mode textarea {
    caret-color: #fff;
}

pre {
    font-family: "Menlo", "Monaco", "Consolas", monospace;
    font-size: 13px;
    line-height: 1.5;
    padding: 12px;
    margin: 0;
    width: 100%;
    min-height: 240px;
    white-space: pre;
    overflow: visible;
    position: absolute;
    top: 0;
    left: 0;
    pointer-events: none;
    color: #333;
}

.dark-mode pre {
    color: #eee;
}

.wrap-text {
    white-space: pre-wrap;
    word-wrap: break-word;
}

/* Popover Menu */
.popover-menu {
    position: absolute;
    top: 100%;
    right: 0;
    margin-top: 8px;
    background: white;
    border: 1px solid #e0e0e0;
    border-radius: 6px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    z-index: 100;
    min-width: 160px;
    padding: 8px 0;
}

.dark-mode .popover-menu {
    background: #2d2d2d;
    border-color: #444;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
}

.popover-item {
    padding: 8px 16px;
    cursor: pointer;
    transition: background-color 0.2s;
}

.popover-item:hover {
    background-color: rgba(0, 0, 0, 0.05);
}

.dark-mode .popover-item:hover {
    background-color: rgba(255, 255, 255, 0.05);
}

.popover-item label {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 13px;
    color: #333;
    cursor: pointer;
}

.dark-mode .popover-item label {
    color: #eee;
}

.popover-item input[type="checkbox"] {
    margin: 0;
}

/* Hidden UI state */
.hidden-ui .header-bar {
    padding: 4px 8px;
    min-height: 30px;
}
</style>
