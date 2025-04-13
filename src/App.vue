<script setup lang="ts">
import { ref, onMounted, watch, nextTick } from "vue";
import tacklet from "tacklet";
import CodeEditor from "./components/CodeEditor.vue";

// Initialize your refs
const code = ref(`// Example function
function example() {
  console.log("Hello, Snipzy!");
  console.log("Tackpad!");
}`);
const darkMode = ref(true);
const wordWrap = ref(false);
const showLineNumbers = ref(true);
const language = ref("javascript");
const isUIHidden = ref(false);

// Connect to Tacklet and load saved data on component mount
onMounted(async () => {
    nextTick(async () => {
        tacklet.on("selected", () => {
            isUIHidden.value = false;
        });
        tacklet.on("deselected", () => {
            isUIHidden.value = true;
        });
        try {
            await tacklet.connect();
            console.log("Connected to tacklet");

            // Get saved data
            const data = await tacklet.getData();

            // Apply saved data if it exists
            if (data) {
                if (data.code !== undefined) code.value = data.code;
                if (data.darkMode !== undefined) darkMode.value = data.darkMode;
                if (data.wordWrap !== undefined) wordWrap.value = data.wordWrap;
                if (data.showLineNumbers !== undefined)
                    showLineNumbers.value = data.showLineNumbers;
                if (data.language !== undefined) language.value = data.language;
            }
            console.log("Data loaded from Tacklet", language.value);
        } catch (error) {
            console.error("Error connecting to Tacklet:", error);
        }
    });
});

// Watch for changes and save data
watch(
    [code, darkMode, wordWrap, showLineNumbers],
    () => {
        try {
            tacklet.setData({
                code: code.value,
                darkMode: darkMode.value,
                wordWrap: wordWrap.value,
                showLineNumbers: showLineNumbers.value,
            });
            console.log("Data saved to Tacklet");
        } catch (error) {
            console.error("Error saving data to Tacklet:", error);
        }
    },
    { deep: true },
);
</script>

<template>
    <div>
        <CodeEditor
            v-model="code"
            v-model:darkMode="darkMode"
            v-model:wordWrap="wordWrap"
            v-model:showLineNumbers="showLineNumbers"
            v-model:language="language"
            v-model:is-hidden="isUIHidden"
        />
    </div>
</template>
<style></style>
