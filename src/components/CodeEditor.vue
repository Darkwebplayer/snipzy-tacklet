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
}>();

const emit = defineEmits<{
    "update:modelValue": [value: string];
    "update:darkMode": [value: boolean];
    "update:wordWrap": [value: boolean];
    "update:showLineNumbers": [value: boolean];
    "update:language": [value: string];
}>();

// Computed properties for settings with getters and setters.
const localDarkMode = computed({
    get: () => props.darkMode,
    set: (value) => emit("update:darkMode", value),
});

const localWordWrap = computed({
    get: () => props.wordWrap,
    set: (value) => emit("update:wordWrap", value),
});

const localShowLineNumbers = computed({
    get: () => props.showLineNumbers,
    set: (value) => emit("update:showLineNumbers", value),
});

const localLanguage = computed({
    get: () => props.language || "javascript",
    set: (value) => emit("update:language", value),
});

const code = ref(props.modelValue);

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

// Cursor tracking reactive variables.
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

// Helper function to update code content and then highlight and update cursor position.
const updateContent = () => {
    if (editorRef.value) {
        const caretOffset = getCaretCharacterOffsetWithin(editorRef.value);

        // ✅ FIX: Convert HTML to plain text with line breaks
        const rawHtml = editorRef.value.innerHTML;
        const textWithLineBreaks = rawHtml
            .replace(/<div>/gi, "<br>")
            .replace(/<\/div>/gi, "")
            .replace(/<br\s*[\/]?>/gi, "\n")
            .replace(/&nbsp;/g, " ")
            .replace(/<[^>]+>/g, ""); // strip tags

        code.value = textWithLineBreaks;
        emit("update:modelValue", code.value);

        requestAnimationFrame(() => {
            highlightCode(caretOffset);
            updateCursorPosition();
        });
    }
};

const updateCode = (event: Event) => {
    updateContent();
};
// Helper function: get caret offset relative to the element's text content.
const getCaretCharacterOffsetWithin = (element: Node): number => {
    let caretOffset = 0;
    const selection = window.getSelection();
    if (selection && selection.rangeCount > 0) {
        const range = selection.getRangeAt(0);
        const preCaretRange = range.cloneRange();
        preCaretRange.selectNodeContents(element);
        preCaretRange.setEnd(range.startContainer, range.startOffset);
        caretOffset = preCaretRange.toString().length;
    }
    return caretOffset;
};

// Helper function: set caret position given an offset.
const setCaretPosition = (element: Node, offset: number) => {
    const range = document.createRange();
    const selection = window.getSelection();
    let currentOffset = offset;
    // Use a tree walker to traverse text nodes.
    const treeWalker = document.createTreeWalker(
        element,
        NodeFilter.SHOW_TEXT,
        null,
    );
    let node: Node | null = null;
    while ((node = treeWalker.nextNode())) {
        const textLength = node.textContent?.length || 0;
        if (currentOffset <= textLength) {
            range.setStart(node, currentOffset);
            range.collapse(true);
            break;
        } else {
            currentOffset -= textLength;
        }
    }
    if (selection) {
        selection.removeAllRanges();
        selection.addRange(range);
    }
};
const highlightCode = (caretOffset?: number) => {
    if (editorRef.value) {
        // ✅ FIX: Convert \n to <br> after syntax highlighting
        const highlighted = hljs.highlight(code.value, {
            language: localLanguage.value,
        });
        editorRef.value.innerHTML = highlighted.value.replace(/\n/g, "<br>");

        if (typeof caretOffset === "number") {
            setTimeout(() => {
                setCaretPosition(editorRef.value!, caretOffset);
            }, 0);
        }
    }
};

// Calculate and update the current cursor position (line and column).
const updateCursorPosition = () => {
    const selection = window.getSelection();
    if (selection && selection.rangeCount > 0 && editorRef.value) {
        const range = selection.getRangeAt(0);
        const preRange = document.createRange();
        preRange.selectNodeContents(editorRef.value);
        preRange.setEnd(range.startContainer, range.startOffset);
        const preText = preRange.toString();
        const lines = preText.split("\n");
        cursorLine.value = lines.length;
        // Adding one so that the column is 1-indexed.
        cursorColumn.value = lines[lines.length - 1].length + 1;
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
    return Array.from(
        { length: code.value.split("\n").length },
        (_, i) => i + 1,
    );
};

watch(
    () => localLanguage.value,
    () => {
        highlightCode();
    },
);

onMounted(() => {
    highlightCode();
    // Update the cursor position on mount.
    updateCursorPosition();
});

// Handle Enter key for inserting a new line.

const handleKeyDown = (event: KeyboardEvent) => {
    const selection = window.getSelection();
    const range = selection?.getRangeAt(0);
    if (event.key === "Tab") {
        event.preventDefault();
        if (range) {
            const tabNode = document.createTextNode("\t");
            range.insertNode(tabNode);
            range.setStartAfter(tabNode);
            range.setEndAfter(tabNode);
            selection?.removeAllRanges();
            selection?.addRange(range);
        }
        updateContent();
    } else if (event.key === "Enter") {
        event.preventDefault();
        if (range) {
            const br = document.createElement("br");
            const spacer = document.createTextNode("\u200B"); // zero-width space
            range.insertNode(br);
            range.setStartAfter(br);
            range.insertNode(spacer);
            range.setStartAfter(spacer);
            range.setEndAfter(spacer);
            selection?.removeAllRanges();
            selection?.addRange(range);
        }
        updateContent();
    } else if (event.key === "Backspace") {
        const { startContainer, startOffset } = range;

        // Case: Inside a text node, at the beginning, and previous sibling is <br>
        if (
            startContainer.nodeType === Node.TEXT_NODE &&
            startOffset === 0 &&
            startContainer.previousSibling &&
            startContainer.previousSibling.nodeName === "BR"
        ) {
            event.preventDefault();
            const br = startContainer.previousSibling;
            const spacer = startContainer;

            const parent = spacer.parentNode;
            br.remove();
            spacer.remove();

            // Set caret correctly after deletion
            if (parent) {
                const newRange = document.createRange();
                const newSelection = window.getSelection();
                // Find the previous sibling where caret should go
                let caretTarget = parent.lastChild;

                // If nothing left, just set to empty text node
                if (!caretTarget || caretTarget.nodeName === "BR") {
                    const fallback = document.createTextNode("");
                    parent.appendChild(fallback);
                    caretTarget = fallback;
                }

                newRange.setStart(
                    caretTarget,
                    caretTarget.textContent?.length || 0,
                );
                newRange.collapse(true);
                if (newSelection) {
                    newSelection.removeAllRanges();
                    newSelection.addRange(newRange);
                }
            }

            updateContent();
            return;
        }

        // Case: In element node just after <br> and spacer
        if (startContainer.nodeType === Node.ELEMENT_NODE && startOffset > 1) {
            const nodeBefore = startContainer.childNodes[startOffset - 1];
            const nodeBeforePrev = startContainer.childNodes[startOffset - 2];

            if (
                nodeBefore?.nodeName === "BR" &&
                nodeBeforePrev?.textContent === "\u200B"
            ) {
                event.preventDefault();

                // Store reference for caret before deletion
                const parent = startContainer;
                const caretTarget = nodeBeforePrev.previousSibling;

                nodeBefore.remove();
                nodeBeforePrev.remove();

                // Set caret after deletion
                if (caretTarget) {
                    const newRange = document.createRange();
                    const newSelection = window.getSelection();
                    newRange.setStart(
                        caretTarget,
                        caretTarget.textContent?.length || 0,
                    );
                    newRange.collapse(true);
                    if (newSelection) {
                        newSelection.removeAllRanges();
                        newSelection.addRange(newRange);
                    }
                }

                updateContent();
                return;
            }
        }
    }
}; // Handle paste event to insert plain text.
const handlePaste = (event: ClipboardEvent) => {
    event.preventDefault();
    const text = event.clipboardData?.getData("text/plain") || "";
    const selection = window.getSelection();
    const range = selection?.getRangeAt(0);
    if (range) {
        range.deleteContents();
        const textNode = document.createTextNode(text);
        range.insertNode(textNode);
        range.setStartAfter(textNode);
        range.setEndAfter(textNode);
        selection?.removeAllRanges();
        selection?.addRange(range);
    }
    updateContent();
};

// Listen to keyup and click events to update cursor position.
const handleCursorEvents = () => {
    updateCursorPosition();
};
</script>

<template>
    <div class="code-editor" :class="{ 'dark-mode': localDarkMode }">
        <div class="toolbar">
            <select v-model="localLanguage" @change="highlightCode">
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
                    <path
                        fill="currentColor"
                        d="M7 6V3a1 1 0 0 1 1-1h12a1 1 0 0 1 1 1v14a1 1 0 0 1-1 1h-3v3c0 .552-.45 1-1.007 1H4.007A1 1 0 0 1 3 21l.003-14c0-.552.45-1 1.006-1zm2 0h8v10h2V4H9z"
                    />
                </svg>
            </button>

            <!-- Display current cursor position -->
            <div class="cursor-position">
                Line: {{ cursorLine }}, Column: {{ cursorColumn }}
            </div>
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
                :class="[
                    `language-${localLanguage}`,
                    { 'wrap-text': localWordWrap },
                ]"
                contenteditable="true"
                @input="updateCode"
                @keydown="handleKeyDown"
                @keyup="handleCursorEvents"
                @click="handleCursorEvents"
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

.cursor-position {
    margin-left: auto;
    font-family: monospace;
    font-size: 14px;
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
