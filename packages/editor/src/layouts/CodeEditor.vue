<template>
  <div ref="codeEditor" class="magic-code-editor"></div>
</template>

<script lang="ts">
import { defineComponent, onMounted, onUnmounted, ref, watch } from 'vue';
import * as monaco from 'monaco-editor';
import serialize from 'serialize-javascript';

const toString = (v: string | any, language: string): string => {
  let value = '';
  if (typeof v !== 'string') {
    value = serialize(v, {
      space: 2,
      unsafe: true,
    }).replace(/"(\w+)":\s/g, '$1: ');
  } else {
    value = v;
  }
  if (language === 'javascript' && value.startsWith('{') && value.endsWith('}')) {
    value = `(${value})`;
  }
  return value;
};

export default defineComponent({
  name: 'magic-code-editor',

  props: {
    initValues: {
      type: [String, Object],
    },

    modifiedValues: {
      type: [String, Object],
    },

    type: {
      type: [String],
      default: () => '',
    },

    language: {
      type: [String],
      default: () => 'javascript',
    },
  },

  emits: ['initd', 'save'],

  setup(props, { emit }) {
    let vsEditor: monaco.editor.IStandaloneCodeEditor | null = null;
    let vsDiffEditor: monaco.editor.IStandaloneDiffEditor | null = null;
    const values = ref('');
    const loading = ref(false);
    const codeEditor = ref<HTMLDivElement>();

    const setEditorValue = (v: string | any, m: string | any) => {
      values.value = toString(v, props.language);

      if (props.type === 'diff') {
        const originalModel = monaco.editor.createModel(values.value, 'text/javascript');
        const modifiedModel = monaco.editor.createModel(toString(m, props.language), 'text/javascript');

        return vsDiffEditor?.setModel({
          original: originalModel,
          modified: modifiedModel,
        });
      }

      return vsEditor?.setValue(values.value);
    };

    const resizeHandler = () => {
      vsEditor?.layout();
      vsDiffEditor?.layout();
    };

    const getEditorValue = () =>
      props.type === 'diff' ? vsDiffEditor?.getModifiedEditor().getValue() : vsEditor?.getValue();

    const init = async () => {
      if (!codeEditor.value) return;

      const options = {
        value: values.value,
        language: props.language,
        tabSize: 2,
        theme: 'vs-dark',
        fontFamily: 'dm, Menlo, Monaco, "Courier New", monospace',
        fontSize: 15,
        formatOnPaste: true,
      };

      if (props.type === 'diff') {
        vsDiffEditor = monaco.editor.createDiffEditor(codeEditor.value, options);
      } else {
        vsEditor = monaco.editor.create(codeEditor.value, options);
      }

      setEditorValue(props.initValues, props.modifiedValues);

      loading.value = false;

      emit('initd', vsEditor);

      codeEditor.value.addEventListener('keydown', (e) => {
        if (e.keyCode === 83 && (navigator.platform.match('Mac') ? e.metaKey : e.ctrlKey)) {
          e.preventDefault();
          emit('save', getEditorValue());
        }
      });

      if (props.type !== 'diff') {
        vsEditor?.onDidBlurEditorWidget(() => {
          emit('save', getEditorValue());
        });
      }

      globalThis.addEventListener('resize', resizeHandler);
    };

    watch(
      () => props.initValues,
      (v, preV) => {
        if (v !== preV) {
          setEditorValue(props.initValues, props.modifiedValues);
        }
      },
      {
        deep: true,
        immediate: true,
      },
    );

    onMounted(async () => {
      loading.value = true;

      init();
    });

    onUnmounted(() => {
      globalThis.removeEventListener('resize', resizeHandler);
    });

    return {
      values,
      loading,
      codeEditor,

      getEditor() {
        return vsEditor || vsDiffEditor;
      },

      setEditorValue,

      focus() {
        vsEditor?.focus();
        vsDiffEditor?.focus();
      },
    };
  },
});
</script>
