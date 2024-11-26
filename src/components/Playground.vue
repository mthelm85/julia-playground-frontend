<template>
  <v-container class="fill-height">
    <v-responsive class="align-centerfill-height mx-auto">
      <v-row>
        <v-col>
          <v-btn color="primary" prepend-icon="mdi-play" @click="runCode" :disabled="isLoading">Run</v-btn>
        </v-col>
      </v-row>
      <v-row>
        <v-col>
          <div id="editor" style="height: 500px;">
            <v-progress-circular v-if="isLoading" indeterminate color="primary" size="60"
              style="position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%); z-index: 10;"></v-progress-circular>
          </div>
          <v-overlay v-model="isLoading" class="align-center justify-center"></v-overlay>
        </v-col>
      </v-row>
    </v-responsive>
    <v-bottom-sheet v-model="drawer">
      <v-card>
        <v-card-title class="d-flex justify-center">Output</v-card-title>
        <v-card-text>
          <pre>{{ output }}</pre>
        </v-card-text>
      </v-card>
    </v-bottom-sheet>
  </v-container>
</template>

<script setup>
import { onMounted, ref } from 'vue';
import * as monaco from 'monaco-editor';
import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker'
import jsonWorker from 'monaco-editor/esm/vs/language/json/json.worker?worker'
import cssWorker from 'monaco-editor/esm/vs/language/css/css.worker?worker'
import htmlWorker from 'monaco-editor/esm/vs/language/html/html.worker?worker'
import tsWorker from 'monaco-editor/esm/vs/language/typescript/ts.worker?worker'

self.MonacoEnvironment = {
  getWorker(_, label) {
    if (label === 'json') {
      return new jsonWorker()
    }
    if (label === 'css' || label === 'scss' || label === 'less') {
      return new cssWorker()
    }
    if (label === 'html' || label === 'handlebars' || label === 'razor') {
      return new htmlWorker()
    }
    if (label === 'typescript' || label === 'javascript') {
      return new tsWorker()
    }
    return new editorWorker()
  }
}

let editor = null;
const drawer = ref(false);
const output = ref('');
const isLoading = ref(false);

onMounted(() => {
  editor = monaco.editor.create(document.getElementById('editor'), {
    value: `# replace this with your code\nprintln("Hello, world!")`,
    language: 'julia',
    theme: 'vs-dark',
    minimap: { enabled: false },
    automaticLayout: true
  });
});

// Clean up the editor when the component is unmounted to avoid memory leaks
onBeforeUnmount(() => {
  if (editor) {
    editor.dispose();
  }
});

function sanitizeCode(code) {
  isLoading.value = true;
  const lines = code.split('\n');
  // Remove any empty lines and lines starting with '#'
  const sanitizedLines = lines.filter(line => line.trim() !== '' && !line.trim().startsWith('#'));

  // Add semicolons to each line except the last one
  for (let i = 0; i < sanitizedLines.length - 1; i++) {
    sanitizedLines[i] = sanitizedLines[i].trim() + ';';
  }

  // Join the lines back into a single string, removing newlines and keeping the last line as is
  return sanitizedLines.join(' ');
}

async function runCode() {
  const code = sanitizeCode(editor.getValue());
  isLoading.value = true;
  editor.updateOptions({ readOnly: true });
  try {
    // Make API call to execute Julia code
    const response = await fetch('http://localhost:5500/execute', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ code: code })
    });

    // if (!response.ok) {
    //   throw new Error('API request failed');
    // }

    const result = await response.json();
    output.value = result.result || result.stderr;
    drawer.value = true;
  } catch (error) {
    console.error('Error executing code:', error);
    output.value = `Error: ${error.message}`;
    drawer.value = true;
  } finally {
    isLoading.value = false;
    editor.updateOptions({ readOnly: false });
  }
}
</script>
