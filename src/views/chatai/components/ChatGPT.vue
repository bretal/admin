<script setup lang="ts">
import "deep-chat";
import { ref, onMounted, computed } from "vue";

const chatRef = ref();

// 从环境变量读取 DeepSeek API Key（仅开发/演示用，生产环境建议用后端代理）
const deepSeekKey = import.meta.env.VITE_DEEPSEEK_API_KEY as string | undefined;
console.log(deepSeekKey, "1111");

// 当配置了 API Key 时使用 DeepSeek 直连，否则使用演示模式
// 注意：system_message 必须为纯英文/ASCII，否则会触发 fetch headers 的 ISO-8859-1 报错
const directConnectionConfig = computed(() => {
  if (!deepSeekKey?.trim()) return undefined;
  return {
    deepSeek: {
      key: deepSeekKey.trim(),
      chat: {
        model: "deepseek-chat",
        system_message:
          "You are a helpful AI assistant. Reply in the same language as the user.",
        max_tokens: 2000,
        temperature: 0.7
      }
    }
  };
});

const isDemoMode = computed(() => !deepSeekKey?.trim());

onMounted(() => {
  if (isDemoMode.value && chatRef.value) {
    chatRef.value.demo = {
      response: () => ({
        text: "仅演示，如需真实 AI 对话，请在 .env.development 中配置 VITE_DEEPSEEK_API_KEY（你的 DeepSeek API Key）。生产环境建议使用后端代理：https://deepchat.dev/docs/connect"
      })
    };
  }
  if (directConnectionConfig.value && chatRef.value) {
    chatRef.value.directConnection = directConnectionConfig.value;
  }
});
</script>

<template>
  <deep-chat
    ref="chatRef"
    style="border-radius: 10px"
    :messageStyles="{
      default: {
        shared: {
          bubble: {
            maxWidth: '100%',
            backgroundColor: 'unset',
            marginTop: '10px',
            marginBottom: '10px'
          }
        },
        user: {
          bubble: {
            marginLeft: '0px',
            color: 'black'
          }
        },
        ai: {
          outerContainer: {
            backgroundColor: 'rgba(247,247,248)',
            borderTop: '1px solid rgba(0,0,0,.1)',
            borderBottom: '1px solid rgba(0,0,0,.1)'
          }
        }
      }
    }"
    :avatars="{
      default: { styles: { position: 'left' } },
      ai: { src: 'https://xiaoxian521.github.io/hyperlink/svg/openai.svg' }
    }"
    :submitButtonStyles="{
      submit: {
        container: {
          default: {
            padding: '1px 0 0 5px',
            backgroundColor: '#19c37d'
          },
          hover: { backgroundColor: '#0bab69' },
          click: { backgroundColor: '#068e56' }
        },
        svg: {
          content:
            '<?xml version=&quot;1.0&quot; ?> <svg viewBox=&quot;0 0 28 28&quot; xmlns=&quot;http://www.w3.org/2000/svg&quot;> <g> <path d=&quot;M21.66,12a2,2,0,0,1-1.14,1.81L5.87,20.75A2.08,2.08,0,0,1,5,21a2,2,0,0,1-1.82-2.82L5.46,13H11a1,1,0,0,0,0-2H5.46L3.18,5.87A2,2,0,0,1,5.86,3.25h0l14.65,6.94A2,2,0,0,1,21.66,12Z&quot;> </path> </g> </svg>',
          styles: {
            default: {
              filter:
                'brightness(0) saturate(100%) invert(100%) sepia(28%) saturate(2%) hue-rotate(69deg) brightness(107%) contrast(100%)'
            }
          }
        }
      },
      loading: {
        container: { default: { backgroundColor: 'white' } },
        svg: {
          styles: {
            default: {
              filter:
                'brightness(0) saturate(100%) invert(72%) sepia(0%) saturate(3044%) hue-rotate(322deg) brightness(100%) contrast(96%)'
            }
          }
        }
      },
      stop: {
        container: {
          default: { backgroundColor: 'white' },
          hover: { backgroundColor: '#dadada52' }
        },
        svg: {
          content:
            '<?xml version=&quot;1.0&quot; encoding=&quot;utf-8&quot;?> <svg viewBox=&quot;0 0 24 24&quot; xmlns=&quot;http://www.w3.org/2000/svg&quot;> <rect width=&quot;24&quot; height=&quot;24&quot; rx=&quot;4&quot; ry=&quot;4&quot; /> </svg>',
          styles: {
            default: {
              width: '0.95em',
              marginTop: '0.32em',
              filter:
                'brightness(0) saturate(100%) invert(72%) sepia(0%) saturate(3044%) hue-rotate(322deg) brightness(100%) contrast(96%)'
            }
          }
        }
      }
    }"
    :textInput="{ placeholder: { text: '发送消息' } }"
    :history="[
      { text: '李白是谁？', role: 'user' },
      {
        text: '李白（701年2月28日－762年），号青莲居士，又号“谪仙人”，是唐代著名的浪漫主义诗人，被后人誉为“诗仙”。',
        role: 'ai'
      }
    ]"
    :demo="isDemoMode"
    :directConnection="directConnectionConfig"
    :connect="{ stream: true }"
  />
</template>
