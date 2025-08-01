<template>
  <div class="vditor-editor-wrapper">
    <!-- 纯文本编辑器 (在纯文本模式下显示) -->
    <textarea
      v-if="isPlainTextMode"
      class="w-full h-[600px] p-4 font-mono text-base border rounded-lg resize-y focus:outline-none focus:ring-2"
      :class="darkMode ? 'bg-gray-800 border-gray-700 text-gray-100 focus:ring-primary-600' : 'bg-white border-gray-300 text-gray-900 focus:ring-primary-500'"
      v-model="plainTextContent"
      :placeholder="$t('markdown.plainTextPlaceholder')"
      @input="syncContentFromPlainText"
    ></textarea>

    <!-- Markdown编辑器 (在Markdown模式下显示) -->
    <div v-else id="vditor" class="w-full border rounded-lg" :class="darkMode ? 'border-gray-700' : 'border-gray-200'"></div>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, watch, nextTick } from "vue";
import { useI18n } from "vue-i18n";

// 懒加载Vditor和CSS
let VditorClass = null;
let vditorCSSLoaded = false;

const loadVditor = async () => {
  if (!VditorClass) {
    await loadVditorCSS();

    // 从assets目录加载Vditor
    const script = document.createElement("script");
    script.src = "/assets/vditor/dist/index.min.js";

    return new Promise((resolve, reject) => {
      script.onload = () => {
        VditorClass = window.Vditor;
        resolve(VditorClass);
      };
      script.onerror = reject;
      document.head.appendChild(script);
    });
  }
  return VditorClass;
};

const loadVditorCSS = async () => {
  if (!vditorCSSLoaded) {
    const link = document.createElement("link");
    link.rel = "stylesheet";
    link.href = "/assets/vditor/dist/index.css";
    document.head.appendChild(link);
    vditorCSSLoaded = true;
  }
};

// 优化的表情配置 - 只包含最常用的表情
const getOptimizedEmojis = () => ({
  // 基本表情 (20个)
  smile: "😊",
  joy: "😂",
  laughing: "😆",
  wink: "😉",
  heart_eyes: "😍",
  thinking: "🤔",
  worried: "😟",
  cry: "😢",
  angry: "😠",
  sunglasses: "😎",
  // 手势表情 (10个)
  thumbsup: "👍",
  thumbsdown: "👎",
  ok_hand: "👌",
  clap: "👏",
  muscle: "💪",
  // 心形表情 (5个)
  heart: "❤️",
  yellow_heart: "💛",
  green_heart: "💚",
  blue_heart: "💙",
  broken_heart: "💔",
  // 符号表情 (10个)
  check: "✅",
  x: "❌",
  warning: "⚠️",
  question: "❓",
  exclamation: "❗",
  star: "⭐",
  fire: "🔥",
  zap: "⚡",
  rocket: "🚀",
  bulb: "💡",
});

const { t } = useI18n();

// Props
const props = defineProps({
  darkMode: {
    type: Boolean,
    default: false,
  },
  isPlainTextMode: {
    type: Boolean,
    default: false,
  },
  modelValue: {
    type: String,
    default: "",
  },
});

// Emits
const emit = defineEmits(["update:modelValue", "editor-ready", "content-change"]);

// 编辑器实例
let editor = null;

// 纯文本内容
const plainTextContent = ref("");
// 原始纯文本内容（保留格式）
const originalPlainTextContent = ref("");

// 初始化编辑器
const initEditor = async () => {
  const vditorContainer = document.getElementById("vditor");
  if (!vditorContainer) {
    console.error("找不到vditor容器元素，无法初始化编辑器");
    return;
  }

  // 开始初始化编辑器

  try {
    // 懒加载Vditor
    const VditorConstructor = await loadVditor();

    const theme = props.darkMode ? "dark" : "light";

    // 检测是否为移动设备
    const isMobile = window.innerWidth <= 768;
    const defaultMode = isMobile ? "ir" : "sv";
    const enableOutline = !isMobile;

    editor = new VditorConstructor("vditor", {
      height: 600,
      minHeight: 400,
      width: "100%",
      mode: defaultMode,
      theme: theme,
      cdn: "/assets/vditor",
      resize: {
        enable: true,
        position: "bottom",
      },
      counter: {
        enable: true,
        type: "text",
      },
      tab: "\t",
      indent: {
        tab: "\t",
        codeBlock: 4,
      },
      preview: {
        delay: 800, // 优化预览延迟
        maxWidth: 800,
        mode: "both",
        theme: theme,
        hljs: {
          lineNumber: true,
          style: props.darkMode ? "vs2015" : "github",
          js: "/assets/vditor/dist/js/highlight.js/third-languages.js",
          css: (style) => `/assets/vditor/dist/js/highlight.js/styles/${style}.min.css`,
        },
        actions: ["desktop", "tablet", "mobile", "mp-wechat", "zhihu"],
        markdown: {
          toc: true,
          mark: true,
          footnotes: true,
          autoSpace: true,
          listStyle: true,
          task: true,
          paragraphBeginningSpace: true,
          fixTermTypo: true,
          media: true,
          mermaid: {
            theme: "default",
            useMaxWidth: false,
          },
        },
        math: {
          engine: "KaTeX",
          inlineDigit: true,
        },
      },
      typewriterMode: true,
      outline: {
        enable: enableOutline,
        position: "left",
      },
      hint: {
        delay: 200,
        emoji: getOptimizedEmojis(),
      },
      toolbar: [
        "emoji",
        "headings",
        "bold",
        "italic",
        "strike",
        "link",
        "|",
        "list",
        "ordered-list",
        "check",
        "outdent",
        "indent",
        "|",
        "quote",
        "line",
        "code",
        "inline-code",
        "insert-before",
        "insert-after",
        "|",
        "table",
        "|",
        "undo",
        "redo",
        "|",
        {
          name: "import-markdown",
          icon: '<svg viewBox="0 0 24 24" width="16" height="16" xmlns="http://www.w3.org/2000/svg"><path fill="currentColor" d="M5,20H19V18H5M19,9H15V3H9V9H5L12,16L19,9Z"></path></svg>',
          tip: t("markdown.toolbar.importFile"),
          click() {
            emit("import-file");
          },
        },
        {
          name: "clear-content",
          icon: '<svg viewBox="0 0 24 24" width="16" height="16" xmlns="http://www.w3.org/2000/svg"><path fill="currentColor" d="M19,4H15.5L14.5,3H9.5L8.5,4H5V6H19M6,19A2,2 0 0,0 8,21H16A2,2 0 0,0 18,19V7H6V19Z"></path></svg>',
          tip: t("markdown.toolbar.clearContent"),
          click() {
            if (confirm(t("markdown.messages.confirmClearContent"))) {
              emit("clear-content");
            }
          },
        },
        {
          name: "copy-formats",
          icon: '<svg viewBox="0 0 24 24" width="16" height="16" xmlns="http://www.w3.org/2000/svg"><path fill="currentColor" d="M19,21H8V7H19M19,5H8A2,2 0 0,0 6,7V21A2,2 0 0,0 8,23H19A2,2 0 0,0 21,21V7A2,2 0 0,0 19,5M16,1H4A2,2 0 0,0 2,3V17H4V3H16V1Z"></path></svg>',
          tip: t("markdown.toolbar.copyFormats"),
          click(event) {
            // 获取按钮位置信息
            const buttonElement = event.target.closest(".vditor-tooltipped");
            if (buttonElement) {
              const rect = buttonElement.getBoundingClientRect();
              emit("show-copy-formats", {
                x: rect.left,
                y: rect.bottom + 5,
              });
            } else {
              emit("show-copy-formats");
            }
          },
        },
        "|",
        "fullscreen",
        "edit-mode",
        "both",
        "outline",
        "preview",
        "export",
        "help",
      ],
      placeholder: t("markdown.editorPlaceholder"),
      cache: {
        enable: true,
        id: "cloudpaste-editor",
        after: (html) => {
          // 缓存后的回调，可以进行一些清理工作
          return html;
        },
      },
      upload: {
        accept: "image/*,.zip,.pdf,.doc,.docx,.xls,.xlsx,.ppt,.pptx",
        token: "",
        linkToImgUrl: "/api/fetch?url=",
        filename(name) {
          return name.replace(/\W/g, "");
        },
      },
      after: () => {
        // 编辑器初始化完成
        emit("editor-ready", editor);
      },
      input: () => {
        try {
          // 确保编辑器完全初始化后再获取内容
          if (editor && editor.getValue && typeof editor.getValue === "function") {
            const content = editor.getValue();
            // 避免重复emit相同内容
            if (content !== lastKnownValue) {
              lastKnownValue = content;
              emit("update:modelValue", content);
              emit("content-change", content);
            }
          }
        } catch (error) {
          console.error("获取编辑器内容时出错:", error);
        }
      },
      customKeymap: {
        Tab: (editor, event) => {
          return false;
        },
      },
    });
  } catch (error) {
    console.error("Vditor编辑器初始化失败:", error);
  }
};

// 安全设置编辑器内容
const safeSetValue = (content) => {
  if (!editor || !editor.setValue || typeof editor.setValue !== "function") return;

  setTimeout(() => {
    if (editor && editor.setValue && typeof editor.setValue === "function") {
      try {
        editor.setValue(content);
      } catch (error) {
        console.error("设置编辑器内容失败:", error);
      }
    }
  }, 500);
};

// 同步纯文本内容
const syncContentFromPlainText = () => {
  originalPlainTextContent.value = plainTextContent.value;
  emit("update:modelValue", plainTextContent.value);
  emit("content-change", plainTextContent.value);
};

// 获取编辑器内容
const getValue = () => {
  if (props.isPlainTextMode) {
    return originalPlainTextContent.value || plainTextContent.value;
  } else if (editor && editor.getValue && typeof editor.getValue === "function") {
    try {
      return editor.getValue();
    } catch (error) {
      console.error("获取编辑器内容时出错:", error);
      return "";
    }
  }
  return "";
};

// 设置编辑器内容
const setValue = (content) => {
  plainTextContent.value = content;
  originalPlainTextContent.value = content;

  if (!props.isPlainTextMode && editor && editor.setValue && typeof editor.setValue === "function") {
    try {
      editor.setValue(content);
    } catch (error) {
      console.error("设置编辑器内容时出错:", error);
    }
  }
};

// 获取HTML内容
const getHTML = () => {
  if (editor && editor.getHTML && typeof editor.getHTML === "function") {
    try {
      return editor.getHTML();
    } catch (error) {
      console.error("获取HTML内容时出错:", error);
      return "";
    }
  }
  return "";
};

// 清空内容
const clearContent = () => {
  if (editor && editor.setValue && typeof editor.setValue === "function") {
    try {
      editor.setValue("");
    } catch (error) {
      console.error("清空编辑器内容时出错:", error);
    }
  }
  plainTextContent.value = "";
  originalPlainTextContent.value = "";
  emit("update:modelValue", "");
  emit("content-change", "");
};

// 监听暗色模式变化
watch(
  () => props.darkMode,
  async (newDarkMode, oldDarkMode) => {
    if (!props.isPlainTextMode && editor && newDarkMode !== oldDarkMode) {
      try {
        let currentValue = "";

        // 安全地获取当前内容
        if (editor && editor.getValue && typeof editor.getValue === "function") {
          try {
            currentValue = editor.getValue();
          } catch (e) {
            console.warn("获取编辑器内容失败，使用空内容:", e);
            currentValue = "";
          }
        }

        // 重新初始化编辑器以应用新主题
        if (editor.destroy) {
          editor.destroy();
        }
        editor = null;

        await initEditor();

        // 设置内容
        if (currentValue) {
          safeSetValue(currentValue);
        }
      } catch (error) {
        console.error("切换主题时出错:", error);
      }
    }
  }
);

// 监听模式切换
watch(
  () => props.isPlainTextMode,
  async (newMode, oldMode) => {
    if (!newMode && oldMode !== newMode) {
      // 切换到Markdown模式
      await nextTick();

      if (editor) {
        try {
          if (editor.destroy) {
            editor.destroy();
          }
        } catch (e) {
          console.error("销毁编辑器时出错:", e);
        }
        editor = null;
      }

      // 初始化编辑器
      const initializeEditor = async () => {
        try {
          await initEditor();

          // 设置内容
          const contentToSet = plainTextContent.value || "";
          if (contentToSet) {
            safeSetValue(contentToSet);
          }
        } catch (error) {
          console.error("初始化编辑器时出错:", error);
        }
      };

      if (window.requestIdleCallback) {
        window.requestIdleCallback(initializeEditor, { timeout: 1000 });
      } else {
        setTimeout(initializeEditor, 100);
      }
    }
  }
);

// 监听内容变化 - 优化性能
let lastKnownValue = "";
watch(
  () => props.modelValue,
  (newValue) => {
    // 避免不必要的getValue()调用
    if (newValue !== lastKnownValue) {
      setValue(newValue);
      lastKnownValue = newValue;
    }
  },
  { immediate: true }
);

// 组件挂载 - 优化性能
onMounted(async () => {
  if (!props.isPlainTextMode) {
    await nextTick();

    // 使用requestIdleCallback优化初始化时机
    const initializeEditor = async () => {
      try {
        await initEditor();

        // 设置初始内容
        if (props.modelValue && editor) {
          safeSetValue(props.modelValue);
        }
      } catch (error) {
        console.error("初始化编辑器时出错:", error);
      }
    };

    if (window.requestIdleCallback) {
      window.requestIdleCallback(initializeEditor, { timeout: 1000 });
    } else {
      // 降级方案
      setTimeout(initializeEditor, 100);
    }
  } else {
    plainTextContent.value = props.modelValue;
    originalPlainTextContent.value = props.modelValue;
  }
});

// 组件卸载
onUnmounted(() => {
  if (!props.isPlainTextMode && editor) {
    try {
      if (editor.destroy && editor.element) {
        editor.destroy();
      }
      editor = null;
    } catch (e) {
      console.warn("销毁编辑器时发生错误:", e);
      editor = null;
    }
  }
});

// 暴露方法
defineExpose({
  getValue,
  setValue,
  getHTML,
  clearContent,
  editor: () => editor,
});
</script>

<style scoped>
.vditor-editor-wrapper {
  width: 100%;
}

/* 纯文本编辑器样式 */
textarea {
  resize: vertical;
  min-height: 400px;
  font-family: Consolas, Monaco, "Andale Mono", "Ubuntu Mono", monospace;
  line-height: 1.6;
  tab-size: 4;
  -moz-tab-size: 4;
  transition: border-color 0.2s, box-shadow 0.2s;
}

textarea:focus {
  outline: none;
}

/* 纯文本编辑器暗色模式 */
textarea.bg-gray-800 {
  color: #d4d4d4;
}
</style>
