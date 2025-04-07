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
}>();

const emit = defineEmits<{
    "update:modelValue": [value: string];
    "update:darkMode": [value: boolean];
    "update:wordWrap": [value: boolean];
    "update:showLineNumbers": [value: boolean];
}>();

// Create a computed property for darkMode with getter and setter
const localDarkMode = computed({
    get: () => props.darkMode,
    set: (value) => emit("update:darkMode", value),
});

// Create a computed property for wordWrap with getter and setter
const localWordWrap = computed({
    get: () => props.wordWrap,
    set: (value) => emit("update:wordWrap", value),
});

// Create a computed property for showLineNumbers with getter and setter
const localShowLineNumbers = computed({
    get: () => props.showLineNumbers,
    set: (value) => emit("update:showLineNumbers", value),
});

const code = ref(props.modelValue);
const language = ref("javascript");

const isWrapped = ref(false);
const showLineNumbers = ref(true);
const editorRef = ref<HTMLPreElement | null>(null);

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

watch(
    () => props.modelValue,
    (newValue) => {
        if (code.value !== newValue) {
            code.value = newValue;
            highlightCode();
        }
    },
);

const updateCode = (event: Event) => {
    const target = event.target as HTMLPreElement;
    code.value = target.textContent || "";
    emit("update:modelValue", code.value);
    requestAnimationFrame(() => {
        highlightCode();
    });
};

const highlightCode = () => {
    if (editorRef.value) {
        const result = hljs.highlight(code.value, { language: language.value });
        editorRef.value.innerHTML = result.value;
        // Restore cursor position
        const selection = window.getSelection();
        if (selection) {
            selection.selectAllChildren(editorRef.value);
            selection.collapseToEnd();
        }
    }
};

const copyCode = async () => {
    try {
        await navigator.clipboard.writeText(code.value);
        alert("Code copied to clipboard!");
    } catch (err) {
        console.error("Failed to copy code:", err);
    }
};

const getLineNumbers = () => {
    const lines = code.value.split("\n").length;
    return Array.from({ length: lines }, (_, i) => i + 1);
};

watch(
    () => language.value,
    () => {
        highlightCode();
    },
);

onMounted(() => {
    highlightCode();
});

const handleKeyDown = (event: KeyboardEvent) => {
    if (event.key === "Enter") {
        event.preventDefault();

        // Get selection
        const selection = window.getSelection();
        const range = selection?.getRangeAt(0);

        // Insert new line
        const textNode = document.createTextNode("\n");
        range?.insertNode(textNode);

        // Move cursor to end
        if (selection && range) {
            range.setStartAfter(textNode);
            range.setEndAfter(textNode);
            selection.removeAllRanges();
            selection.addRange(range);
        }

        // Update code and highlight
        if (editorRef.value) {
            code.value = editorRef.value.textContent || "";
            emit("update:modelValue", code.value);
            requestAnimationFrame(() => {
                highlightCode();
            });
        }
    }
};
const handlePaste = (event: ClipboardEvent) => {
    event.preventDefault();

    // Get plain text from clipboard
    const text = event.clipboardData?.getData("text/plain") || "";

    // Insert text at cursor position
    const selection = window.getSelection();
    const range = selection?.getRangeAt(0);

    if (range) {
        range.deleteContents();
        const textNode = document.createTextNode(text);
        range.insertNode(textNode);

        // Move cursor to end of pasted text
        range.setStartAfter(textNode);
        range.setEndAfter(textNode);
        selection?.removeAllRanges();
        selection?.addRange(range);
    }

    // Update code and highlight
    if (editorRef.value) {
        code.value = editorRef.value.textContent || "";
        emit("update:modelValue", code.value);
        requestAnimationFrame(() => {
            highlightCode();
        });
    }
};
</script>

<template>
    <div class="code-editor" :class="{ 'dark-mode': darkMode }">
        <div class="toolbar">
            <select v-model="language" @change="highlightCode">
                <option v-for="lang in languages" :key="lang" :value="lang">
                    {{ lang }}
                </option>
            </select>

            <label>
                <input type="checkbox" v-model="localDarkMode" />
                Dark Mode
            </label>

            <label>
                <input type="checkbox" v-model="localWordWrap" />
                Wrap Text
            </label>

            <label>
                <input type="checkbox" v-model="localShowLineNumbers" />
                Show Line Numbers
            </label>

            <button @click="copyCode" title="copy code">
                <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="16"
                    height="16"
                    viewBox="0 0 24 24"
                >
                    <!-- Icon from Remix Icon by Remix Design - https://github.com/Remix-Design/RemixIcon/blob/master/License -->
                    <path
                        fill="currentColor"
                        d="M7 6V3a1 1 0 0 1 1-1h12a1 1 0 0 1 1 1v14a1 1 0 0 1-1 1h-3v3c0 .552-.45 1-1.007 1H4.007A1 1 0 0 1 3 21l.003-14c0-.552.45-1 1.006-1zm2 0h8v10h2V4H9z"
                    />
                </svg>
            </button>
        </div>

        <div
            class="editor-container"
            :class="{ 'with-line-numbers': localShowLineNumbers }"
        >
            <div v-if="localShowLineNumbers" class="line-numbers">
                <div
                    v-for="num in getLineNumbers()"
                    :key="num"
                    class="line-number"
                >
                    {{ num }}
                </div>
            </div>
            <pre
                ref="editorRef"
                :class="[`language-${language}`, { 'wrap-text': wordWrap }]"
                contenteditable="true"
                @input="updateCode"
                @keydown="handleKeyDown"
                @paste="handlePaste"
                spellcheck="false"
                >{{ code }}</pre
            >
        </div>
    </div>
</template>

<style scoped>
.code-editor {
    border: 1px solid #ddd;
    border-radius: 4px;
    overflow: hidden;
    background: #fff;
    text-align: left;
    color: #333;
}

.dark-mode {
    background: #1e1e1e;
    color: #fff;
}

.dark-mode .toolbar {
    background: #333;
    color: #fff;
}

.toolbar {
    padding: 10px;
    background: #f5f5f5;
    border-bottom: 1px solid #ddd;
    display: flex;
    gap: 15px;
    align-items: center;
    color: #333;
}

.toolbar select,
.toolbar button {
    padding: 4px 8px;
    border: 1px solid #ddd;
    border-radius: 4px;
    background: white;
    color: #333;
}

.dark-mode .toolbar select,
.dark-mode .toolbar button {
    background: #2d2d2d;
    color: #fff;
    border-color: #444;
}

.editor-container {
    display: flex;
    position: relative;
}

.line-numbers {
    padding: 5px 10px;
    background: #f5f5f5;
    border-right: 1px solid #ddd;
    color: #999;
    user-select: none;
    text-align: right;
    min-width: 40px;
}

.dark-mode .line-numbers {
    background: #2d2d2d;
    border-right-color: #444;
}

.line-number {
    font-family: monospace;
    font-size: 14px;
    line-height: 1.5;
}

pre {
    margin: 0;
    padding: 15px;
    outline: none;
    white-space: pre;
    overflow-x: auto;
    min-height: 200px;
    flex-grow: 1;
    font-family: monospace;
    font-size: 14px;
    line-height: 1.5;
    text-align: left;
    color: #333;
}

.dark-mode pre {
    color: #fff;
}

.wrap-text {
    white-space: pre-wrap;
    word-wrap: break-word;
}

button {
    cursor: pointer;
}

button:hover {
    opacity: 0.9;
}

:deep(.hljs) {
    background: transparent !important;
    padding: 0 !important;
}

select {
    color: #333 !important;
}

.dark-mode select {
    color: #fff !important;
}
</style>
