<!-- 每个回话对应的聊天内容 -->
<script setup lang="ts">
import type { AnyObject } from "typescript-api-pro";
import type { BubbleProps } from "vue-element-plus-x/types/Bubble";
import type { BubbleListInstance } from "vue-element-plus-x/types/BubbleList";
import type { FilesCardProps } from "vue-element-plus-x/types/FilesCard";
import type { ThinkingStatus } from "vue-element-plus-x/types/Thinking";
import { useHookFetch } from "hook-fetch/vue";
import { Sender } from "vue-element-plus-x";
import { useRoute } from "vue-router";
import { send } from "@/api";
import { getKnowledgeList, getWorkflowList } from "@/api/chat";
import FilesSelect from "@/components/FilesSelect/index.vue";
import ModelSelect from "@/components/ModelSelect/index.vue";
import { useChatStore } from "@/stores/modules/chat";
import { useFilesStore } from "@/stores/modules/files";
import { useModelStore } from "@/stores/modules/model";
import { useSessionStore } from "@/stores/modules/session";
import { useUserStore } from "@/stores/modules/user";
import { codeXRender } from "@/utils/markdownRenderers";

type MessageItem = BubbleProps & {
  key: number;
  role: "ai" | "user" | "system";
  avatar: string;
  thinkingStatus?: ThinkingStatus;
  thinlCollapse?: boolean;
  reasoning_content?: string;
};
const copyIconMap = ref<Record<number, string>>({}); // 记录每条消息的复制按钮图标
const editingMessageKeys = ref<number[]>([]); // 跟踪多个编辑中的消息
const editedContents = ref<Record<number, string>>({}); // 存储每条消息的临时编辑内容

const copyIcon = ref("CopyDocument");
const route = useRoute();
const chatStore = useChatStore();
const modelStore = useModelStore();
const filesStore = useFilesStore();
const sessionStore = useSessionStore();
const userStore = useUserStore();

// 用户头像
const avatar = computed(() => {
  const userInfo = userStore.userInfo;
  return userInfo?.avatar || "https://avatars.githubusercontent.com/u/32251822?s=96&v=4";
});

const inputValue = ref("");
const senderRef = ref<InstanceType<typeof Sender> | null>(null);
const bubbleItems = ref<MessageItem[]>([]);
const bubbleListRef = ref<BubbleListInstance | null>(null);

// 推理和联网开关状态
const isReasoningEnabled = ref(false);
const isWebSearchEnabled = ref(false);

// 知识库列表配置
const knowledgeList = ref<any[]>([]);

const workflowList = ref<any[]>([]);

// 知识库弹窗状态
const knowledgePopoverRef = ref();
const isKnowledgePopoverVisible = ref(false);
const selectedKnowledgeId = ref<string>("");
const selectedKnowledgeName = ref<string>("知识库");
const isWorkflowVisible = ref(false);
const selectedWorkflowName = ref<string>("工作流");

const workflowParams = ref<AnyObject>({
  pageSize: 10,
  currentPage: 1,
});

const workFlowRunner = ref<AnyObject>({});
const reSumeRunner = ref<AnyObject>({});

// 是否正在加载
const isWorkflowLoading = ref(false);
// 是否还有更多数据
const hasMoreWorkflows = ref(true);

const isResume = ref(false);

// 跟踪当前节点 ID
let currentNodeId: string | null = null;

// 处理 NODE_CHUNK 事件的函数
function handleNodeChunk(data: any, isLastChunk = false) {
  const nodeId = data.nodeId; // 从数据中提取节点 ID
  const content = data.content; // 从数据中提取内容

  console.log("nodeId-nodeId", nodeId, currentNodeId);

  console.log("content", content);

  // 判断节点 ID 是否发生变化

  if (nodeId && nodeId !== currentNodeId) {
    // 创建一个新的系统消息气泡
    if (currentNodeId !== null) {
      const lastMessage = bubbleItems.value[bubbleItems.value.length - 1];
      if (lastMessage) {
        lastMessage.thinkingStatus = "end";
        lastMessage.loading = false;
      }
    }
    if (content && currentNodeId !== null) {
      addMessage("", false); // 添加一个空的系统消息气泡
    }
    currentNodeId = nodeId; // 更新当前节点 ID
  }

  if (nodeId && content) {
    // 将内容追加到最新的气泡中
    const lastMessage = bubbleItems.value[bubbleItems.value.length - 1];
    if (lastMessage && lastMessage.role === "system") {
      lastMessage.content += content;
      lastMessage.thinkingStatus = "end";
      lastMessage.loading = false;
    }
  }

  if (isLastChunk) {
    //  如果是最后一块数据，清理空白气泡
    const lastMessage = bubbleItems.value[bubbleItems.value.length - 1];
    if (lastMessage && lastMessage.role === "system" && !lastMessage.content.trim()) {
      // 删除最后一个空白气泡
      bubbleItems.value.pop();
    }

    if (lastMessage) {
      lastMessage.thinkingStatus = "end";
      lastMessage.loading = false;
    }
  }

  if (isResume.value) {
    const lastMessage = bubbleItems.value[bubbleItems.value.length - 1];
    lastMessage.thinkingStatus = "end";
    lastMessage.loading = false;
    isLoading.value = false;
  }
}

function chooseWorkflowItem(item: any) {
  isWorkflowVisible.value = true;
  selectedWorkflowName.value = item.title;
  workFlowRunner.value.uuid = item.uuid;
  let nodes = [...item.nodes];
  let user_inputs = nodes[0].inputConfig.user_inputs[0];
  let inputsObj = {
    uuid: nodes[0].uuid,
    name: user_inputs.name,
    required: user_inputs.required,
    content: {
      title: user_inputs.title,
      value: "",
      type: user_inputs.type,
    },
  };

  workFlowRunner.value.inputs = [inputsObj];

  console.log("workFlowRunner", workFlowRunner.value);
}

// 监听滚动事件
function handleScroll(event: Event) {
  const target = event.target as HTMLElement;
  const { scrollTop, scrollHeight, clientHeight } = target;

  // 判断是否滚动到底部
  if (
    scrollTop + clientHeight >= scrollHeight - 10 &&
    !isWorkflowLoading.value &&
    hasMoreWorkflows.value
  ) {
    loadWorkflowList(true); // 加载更多
  }
}

// 加载工作流列表
async function loadWorkflowList(isLoadMore = false) {
  if (isWorkflowLoading.value || !hasMoreWorkflows.value) return; // 防止重复请求或无数据时继续加载
  isWorkflowLoading.value = true;
  try {
    const response = await getWorkflowList(workflowParams.value);
    console.log("工作流列表:", response);
    if (response?.data && response.data?.records && Array.isArray(response.data.records)) {
      const newRecords = response.data.records;

      if (isLoadMore) {
        // 追加数据
        workflowList.value = [...workflowList.value, ...newRecords];
      } else {
        // 替换数据（首次加载）
        workflowList.value = newRecords;
      }

      // 更新分页参数
      workflowParams.value.currentPage += 1;

      // 判断是否还有更多数据
      hasMoreWorkflows.value = response.data.total > workflowList.value.length;
    } else {
      // 如果返回数据为空或格式不正确，标记为无更多数据
      hasMoreWorkflows.value = false;
    }
  } catch (error) {
    console.error("Failed to load workflow list:", error);
    hasMoreWorkflows.value = false; // 出错时也停止加载
  } finally {
    isWorkflowLoading.value = false;
  }
}
// 加载知识库列表
async function loadKnowledgeList() {
  try {
    const response = await getKnowledgeList();
    if (response?.rows && Array.isArray(response.rows)) {
      knowledgeList.value = response.rows.map((item: any) => ({
        id: item.id,
        name: item.name,
        icon: "Document",
      }));
    }
  } catch (error) {
    console.error("Failed to load knowledge list:", error);
  }
}

// 插入知识库标签
function insertKnowledgeTag(knowledgeId: string) {
  const knowledge = knowledgeList.value.find((k) => k.id === knowledgeId);
  if (knowledge) {
    selectedKnowledgeId.value = knowledgeId;
    selectedKnowledgeName.value = knowledge.name;
    chatStore.setKnowledgeId(knowledgeId);
    // 关闭弹窗
    knowledgePopoverRef.value?.hide();
  }
}

// 清除知识库选择
function clearKnowledgeSelection() {
  selectedKnowledgeId.value = "";
  selectedKnowledgeName.value = "知识库";
  chatStore.setKnowledgeId("");
}

// 从 localStorage 恢复推理状态
onMounted(async () => {
  bubbleItems.value.forEach((item) => {
    copyIconMap.value[item.key] = "CopyDocument";
  });
  const enableThinking = localStorage.getItem("enableThinking");
  if (enableThinking === "true") {
    isReasoningEnabled.value = true;
    localStorage.removeItem("enableThinking");
  }
  const enableInternet = localStorage.getItem("enableInternet");
  if (enableInternet === "true") {
    isWebSearchEnabled.value = true;
    localStorage.removeItem("enableInternet");
  }
  const isWorkflow = localStorage.getItem("isWorkflowVisible");
  if (isWorkflow === "true") {
    isWorkflowVisible.value = true;
    localStorage.removeItem("isWorkflowVisible");
  }

  const workFlowRunnerStr = localStorage.getItem("workFlowRunner");
  if (workFlowRunnerStr) {
    const workFlowRunnerObj = JSON.parse(workFlowRunnerStr);
    workFlowRunner.value = { ...workFlowRunnerObj };
    localStorage.removeItem("workFlowRunner");
  }

  const selectedWorkflowNameStr = localStorage.getItem("selectedWorkflowName");
  if (selectedWorkflowNameStr) {
    selectedWorkflowName.value = selectedWorkflowNameStr;
    localStorage.removeItem("selectedWorkflowName");
  }

  // 加载知识库列表
  await loadKnowledgeList();

  await loadWorkflowList();

  // 从 store 中同步知识库选择状态
  if (chatStore.knowledgeId) {
    const knowledge = knowledgeList.value.find((k) => k.id === chatStore.knowledgeId);
    if (knowledge) {
      selectedKnowledgeId.value = chatStore.knowledgeId;
      selectedKnowledgeName.value = knowledge.name;
    }
  }
});

const {
  stream,
  loading: isLoading,
  cancel,
} = useHookFetch({
  request: send,
  onError: (err) => {
    console.warn("测试错误拦截", err);
  },
});
// 记录进入思考中
let isThinking = false;

watch(
  () => route.params?.id,
  async (_id_) => {
    if (_id_) {
      if (_id_ !== "not_login") {
        // 判断的当前会话id是否有聊天记录，有缓存则直接赋值展示
        if (chatStore.chatMap[`${_id_}`] && chatStore.chatMap[`${_id_}`].length) {
          bubbleItems.value = chatStore.chatMap[`${_id_}`] as MessageItem[];
          // 滚动到底部
          setTimeout(() => {
            bubbleListRef.value?.scrollToBottom();
          }, 350);
          return;
        }

        // 无缓存则请求聊天记录
        await chatStore.requestChatList(`${_id_}`);
        // 请求聊天记录后，赋值回显，并滚动到底部
        bubbleItems.value = chatStore.chatMap[`${_id_}`] as MessageItem[];

        // 滚动到底部
        setTimeout(() => {
          bubbleListRef.value?.scrollToBottom();
        }, 350);
      }

      // 如果本地有发送内容 ，则直接发送
      const v = localStorage.getItem("chatContent");
      if (v) {
        // 发送消息
        setTimeout(() => {
          startSSE(v);
        }, 350);

        localStorage.removeItem("chatContent");
      }
    }
  },
  { immediate: true, deep: true },
);

// 封装数据处理逻辑
function handleDataChunk(chunk: AnyObject) {
  console.log("isResume", isResume.value);
  try {
    // 新的 SSE 格式：data 字段直接包含内容
    let messageData = chunk.data;

    // 如果 chunk 本身就是从 result 中解出来的，可能需要直接用 chunk
    if (!messageData && chunk.content) {
      messageData = chunk.content;
    }

    if (!messageData) {
      return;
    }

    // 处理不同的内容格式
    let contentToAdd = "";
    let reasoningToAdd = "";

    // 如果是字符串，直接处理
    if (typeof messageData === "string") {
      contentToAdd = messageData;
    }
    // 如果是对象，提取 content 和 reasoning_content
    else if (typeof messageData === "object") {
      reasoningToAdd = messageData.reasoning_content || "";
      contentToAdd = messageData.content || "";
    }

    // 处理推理内容
    if (reasoningToAdd) {
      const lastMsg = bubbleItems.value[bubbleItems.value.length - 1];
      lastMsg.thinkingStatus = "thinking";
      lastMsg.loading = true;
      lastMsg.thinlCollapse = true;
      if (bubbleItems.value.length) {
        bubbleItems.value[bubbleItems.value.length - 1].reasoning_content += reasoningToAdd;
      }
    }

    // 处理回复内容（包括 <think></think> 标签格式）
    if (contentToAdd) {
      let currentText = contentToAdd;
      const lastMessage = bubbleItems.value[bubbleItems.value.length - 1];

      // 1. 处理 <think> 标签之前的内容
      if (!isThinking && currentText.includes("<think>")) {
        const thinkIdx = currentText.indexOf("<think>");
        if (thinkIdx > 0) {
          const beforeThink = currentText.substring(0, thinkIdx);
          lastMessage.content += beforeThink;
        }
        currentText = currentText.substring(thinkIdx + 7); // 移除 <think>
        isThinking = true;
        lastMessage.thinkingStatus = "thinking";
        lastMessage.loading = true;
        lastMessage.thinlCollapse = true;
      }

      // 2. 处理 </think> 标签及之前的内容
      if (isThinking && currentText.includes("</think>")) {
        const thinkEndIdx = currentText.indexOf("</think>");
        if (thinkEndIdx > 0) {
          const thinkContent = currentText.substring(0, thinkEndIdx);
          lastMessage.reasoning_content += thinkContent;
        }
        currentText = currentText.substring(thinkEndIdx + 8); // 移除 </think>
        isThinking = false;
        lastMessage.thinkingStatus = "end";
        lastMessage.loading = false;
      }

      // 3. 处理剩余内容（思考过程中未完成的部分 或 思考之后的部分）
      if (currentText) {
        if (isThinking) {
          // 还在思考模式中，内容属于推理
          lastMessage.reasoning_content += currentText;
        } else {
          if (isResume.value) {
            lastMessage.thinkingStatus = "end";
            lastMessage.loading = false;
            isLoading.value = false;
          }
          // 已结束思考模式，内容属于最终回复
          lastMessage.content += currentText;
        }
      }
    }
  } catch (err) {
    console.error("解析数据时出错:", err);
  }
}

// 封装错误处理逻辑
function handleError(err: any) {
  console.error("Fetch error:", err);
}

async function startSSE(chatContent: string) {
  currentNodeId = null; // 每次发送消息前先置空存储的nodeId 防止多余系统气泡生成
  if (isWorkflowVisible.value && !workFlowRunner.value.hasOwnProperty("inputs")) {
    ElMessage.error("请选择工作流！");
    return;
  }
  try {
    // 添加用户输入的消息
    inputValue.value = "";
    addMessage(chatContent, true);
    addMessage("", false);

    // 这里有必要调用一下 BubbleList 组件的滚动到底部 手动触发 自动滚动
    bubbleListRef.value?.scrollToBottom();

    // 获取最后一条用户消息（后端做了长期记忆缓存，只需发送最新的用户消息）
    const lastUserMessage = bubbleItems.value.filter((item: any) => item.role === "user").pop();

    // 处理工作流模式下json数据拼接
    if (isWorkflowVisible.value) {
      workFlowRunner.value.inputs[0].content.value = chatContent;
    }

    if (isResume.value) {
      reSumeRunner.value.feedbackContent = chatContent;
    }

    for await (const chunk of stream({
      messages: lastUserMessage
        ? [
            {
              role: lastUserMessage.role,
              content: lastUserMessage.content,
            },
          ]
        : [],
      sessionId: route.params?.id !== "not_login" ? String(route.params?.id) : undefined,
      userId: userStore.userInfo?.userId,
      model: modelStore.currentModelInfo.modelName ?? "",
      enableThinking: isReasoningEnabled.value,
      enableInternet: isWebSearchEnabled.value,
      knowledgeId: chatStore.knowledgeId || undefined,
      enableWorkFlow: isWorkflowVisible.value,
      workFlowRunner: workFlowRunner.value,
      isResume: isResume.value,
      reSumeRunner: reSumeRunner.value,
    })) {
      // 提取原始数据
      const rawData = chunk.result || chunk.source;
      // 处理连接开始事件
      if (rawData === ":connected") {
        continue;
      }

      // 处理连接结束事件
      if (rawData === ":disconnected") {
        break;
      }
      if (typeof rawData === "string" && rawData.includes("DONE") && rawData.includes("data:")) {
        isResume.value = false;
        reSumeRunner.value = {};
        // 判断是否是最后一块数据
        handleNodeChunk({}, true);
      }
      if (
        typeof rawData === "string" &&
        rawData.includes("NODE_WAIT_FEEDBACK_BY") &&
        rawData.includes("data:")
      ) {
        // 判断是否是最后一块数据
        handleNodeChunk({}, true);
      }

      if (typeof rawData === "string" && rawData.includes("START") && rawData.includes("data:")) {
        isResume.value = true;
        const dataMatch = rawData.match(/data:([\s\S]*?)(?=\nevent:|$)/);
        let dataStr = dataMatch?.[1]?.trim();
        let data = dataStr ? JSON.parse(dataStr) : null;
        reSumeRunner.value.runtimeUuid = data.uuid;
      }
      if (isWorkflowVisible.value) {
        if (
          typeof rawData === "string" &&
          (rawData.includes("NODE_CHUNK") || rawData.includes("NODE_WAIT_FEEDBACK_BY")) &&
          rawData.includes("data:")
        ) {
          const eventMatch = rawData.match(/event:([\s\S]*?)\ndata:/);
          const event = eventMatch ? eventMatch[1] : null;

          let nodeUuid = "";
          if (event.startsWith("[NODE_CHUNK_")) {
            nodeUuid = event.replace("[NODE_CHUNK_", "").replace("]", "");
          }
          if (event.startsWith("[NODE_WAIT_FEEDBACK_BY_")) {
            nodeUuid = event.replace("[NODE_WAIT_FEEDBACK_BY_", "").replace("]", "");
          }

          // 提取 data 字段的内容
          const dataMatch = rawData.match(/data:([\s\S]*?)(?=\nevent:|$)/);
          let data = dataMatch?.[1]?.trim();
          const showData = {
            nodeId: nodeUuid,
            content: data,
          };
          if (data) {
            data = data
              .split("\ndata:")
              .map((line) => line.trim())
              .filter((line) => line !== "")
              .join("\n");
          }

          // 判断是否是最后一块数据
          const isLastChunk = rawData.includes(":disconnected");

          // 调用 handleNodeChunk
          handleNodeChunk(showData, isLastChunk);
        }
      } else {
        if (
          typeof rawData === "string" &&
          rawData.includes("event:") &&
          rawData.includes("data:")
        ) {
          // 提取 event 类型
          const eventMatch = rawData.match(/event:(\w+)/);
          const event = eventMatch?.[1];
          const dataMatch = rawData.match(/data:([\s\S]*?)(?=\nevent:|$)/);
          let data = dataMatch?.[1]?.trim();

          // 清理 data 中可能包含的多余 data: 前缀（当有多行 data: 时）
          if (data) {
            data = data
              .split("\ndata:")
              .map((line) => line.trim())
              .filter((line) => line !== "")
              .join("\n");
          }

          // 只有当 data 不为空且不是格式错误的 'data:' 字符串时才处理

          if (event === "message" && data && data.length > 0 && data !== "data:") {
            handleDataChunk({ data });
          }
        }
      }
    }
  } catch (err) {
    handleError(err);
  } finally {
    // 停止打字器状态
    if (bubbleItems.value.length) {
      const lastMessage = bubbleItems.value[bubbleItems.value.length - 1];
      lastMessage.typing = false;
      // 无条件重置 loading（停止打字动画）
      lastMessage.loading = false;
      // 重置思考状态：如果还在思考中，标记为已完成
      if (lastMessage.thinkingStatus === "thinking") {
        lastMessage.thinkingStatus = "end";
      }
      // 重置isThinking标志
      isThinking = false;
    }
  }
}

// 中断请求
async function cancelSSE() {
  cancel();
  // 结束最后一条消息打字状态
  if (bubbleItems.value.length) {
    bubbleItems.value[bubbleItems.value.length - 1].typing = false;
  }
}
function copyToClipboard(text: string, key: number) {
  navigator.clipboard
    .writeText(text)
    .then(() => {
      // 更新当前消息的图标为打钩
      copyIconMap.value[key] = "Check";

      // 延迟恢复原始图标
      setTimeout(() => {
        copyIconMap.value[key] = "CopyDocument";
      }, 2000);
    })
    .catch((err) => {
      console.error("复制失败:", err);
      ElMessage.error("复制失败，请手动复制");
    });
}

// 添加消息 - 维护聊天记录
function addMessage(message: string, isUser: boolean) {
  const i = bubbleItems.value.length;
  const obj: MessageItem = {
    key: i,
    avatar: isUser
      ? avatar.value
      : "https://cube.elemecdn.com/0/88/03b0d39583f48206768a7534e55bcpng.png",
    avatarSize: "32px",
    role: isUser ? "user" : "system",
    placement: isUser ? "end" : "start",
    isMarkdown: !isUser,
    loading: !isUser,
    content: message || "",
    reasoning_content: "",
    thinkingStatus: "start",
    thinlCollapse: false,
    noStyle: !isUser,
  };
  bubbleItems.value.push(obj);
}

// 展开收起 事件展示
function handleChange(_payload: { value: boolean; status: ThinkingStatus }) {}

function handleDeleteCard(_item: FilesCardProps, index: number) {
  filesStore.deleteFileByIndex(index);
}

// 编辑消息
function startEditing(item: MessageItem) {
  if (!editingMessageKeys.value.includes(item.key)) {
    editingMessageKeys.value.push(item.key); // 将消息 key 加入编辑列表
    editedContents.value[item.key] = item.content; // 初始化编辑内容
  }
  // ⭐ 关键：关闭 Bubble 样式
  item.noStyle = true;
  item.class = "editing-bubble"; // ⭐ 新增
}

// Cancel editing and revert to original content
function cancelEditingByKey(key: number) {
  const item = bubbleItems.value.find((i) => i.key === key);
  if (item) {
    item.noStyle = false; // 恢复气泡
    item.class = "";
  }
  editingMessageKeys.value = editingMessageKeys.value.filter((k) => k !== key);
  delete editedContents.value[key];
}

// Send edited message as a new chat message
function sendMessageByKey(key: number) {
  const newContent = editedContents.value[key];
  if (newContent) {
    // addMessage(newContent, true);
    startSSE(newContent);
    cancelEditingByKey(key);
  }
}
function handleCreateNewChat() {
  sessionStore.createSessionBtn();
}

watch(
  () => filesStore.filesList.length,
  (val) => {
    if (val > 0) {
      nextTick(() => {
        senderRef.value?.openHeader();
      });
    } else {
      nextTick(() => {
        senderRef.value?.closeHeader();
      });
    }
  },
);
</script>

<template>
  <div class="chat-with-id-container">
    <div class="chat-warp">
      <!-- 默认测试图表 - 始终显示在头部 -->
      <!-- <div class="default-chart-wrapper">
        <EchartsRenderer
          :selfProps="{
            code: DEFAULT_CHART_CONFIG,
            width: '100%',
            height: '500px',
            theme: 'dark',
            title: '✅ ECharts 组件状态正常 - 可以接收后端图表数据'
          }"
        />
      </div> -->

      <BubbleList ref="bubbleListRef" :list="bubbleItems" max-height="calc(100vh - 240px)">
        <template #header="{ item }">
          <Thinking
            v-if="item.reasoning_content"
            v-model="item.thinlCollapse"
            :content="item.reasoning_content"
            :status="item.thinkingStatus"
            class="thinking-chain-warp"
            @change="handleChange"
          />
        </template>

        <template #content="{ item }">
          <!-- chat 内容走 markdown -->
          <XMarkdown
            v-if="item.content && item.role === 'system'"
            :markdown="item.content"
            :code-x-render="codeXRender"
            class="markdown-body"
            :themes="{ light: 'github-light', dark: 'github-dark' }"
            default-theme-mode="dark"
          />
          <!-- user 内容 纯文本 -->
          <div v-if="item.content && item.role === 'user'" class="userContent">
            <div class="user-bubble" :class="{ editing: editingMessageKeys.includes(item.key) }">
              <template v-if="!editingMessageKeys.includes(item.key)">
                <div class="user-content">
                  {{ item.content }}
                </div>
              </template>

              <template v-else>
                <div class="edit-card">
                  <el-input
                    v-model="editedContents[item.key]"
                    type="textarea"
                    autosize
                    class="edit-input"
                  />
                  <div class="edit-actions">
                    <el-button size="small" @click="cancelEditingByKey(item.key)"> 取消 </el-button>
                    <el-button type="primary" size="small" @click="sendMessageByKey(item.key)">
                      发送
                    </el-button>
                  </div>
                </div>
              </template>
            </div>

            <div v-if="!editingMessageKeys.includes(item.key)" class="copy-button-container">
              <el-tooltip content="复制" placement="bottom">
                <el-button
                  class="copy-btn"
                  :icon="copyIconMap[item.key] || 'CopyDocument'"
                  size="small"
                  @click="copyToClipboard(item.content, item.key)"
                />
              </el-tooltip>
              <el-tooltip content="编辑" placement="bottom">
                <el-button class="copy-btn" icon="Edit" size="small" @click="startEditing(item)" />
              </el-tooltip>
            </div>
          </div>
        </template>
      </BubbleList>

      <div class="sender-wrapper">
        <!-- 新对话按钮 - 紧贴输入框左上角 -->
        <div class="new-chat-btn" @click="handleCreateNewChat">
          <el-icon class="btn-icon">
            <Plus />
          </el-icon>
          <span class="btn-text">新对话</span>
        </div>

        <Sender
          ref="senderRef"
          v-model="inputValue"
          class="chat-defaul-sender"
          :auto-size="{
            maxRows: 6,
            minRows: 2,
          }"
          variant="updown"
          clearable
          allow-speech
          :loading="isLoading"
          @submit="startSSE"
          @cancel="cancelSSE"
        >
          <template #header>
            <div class="sender-header p-12px pt-6px pb-0px">
              <Attachments
                :items="filesStore.filesList"
                :hide-upload="true"
                @delete-card="handleDeleteCard"
              >
                <template #prev-button="{ show, onScrollLeft }">
                  <div
                    v-if="show"
                    class="prev-next-btn left-8px flex-center w-22px h-22px rounded-8px border-1px border-solid border-[rgba(0,0,0,0.08)] c-[rgba(0,0,0,.4)] hover:bg-#f3f4f6 bg-#fff font-size-10px"
                    @click="onScrollLeft"
                  >
                    <el-icon>
                      <ArrowLeftBold />
                    </el-icon>
                  </div>
                </template>

                <template #next-button="{ show, onScrollRight }">
                  <div
                    v-if="show"
                    class="prev-next-btn right-8px flex-center w-22px h-22px rounded-8px border-1px border-solid border-[rgba(0,0,0,0.08)] c-[rgba(0,0,0,.4)] hover:bg-#f3f4f6 bg-#fff font-size-10px"
                    @click="onScrollRight"
                  >
                    <el-icon>
                      <ArrowRightBold />
                    </el-icon>
                  </div>
                </template>
              </Attachments>
            </div>
          </template>
          <template #prefix>
            <div class="flex-1 flex items-center gap-8px flex-none w-fit overflow-hidden">
              <FilesSelect />
              <ModelSelect />

              <!-- 知识库选择下拉菜单 -->
              <el-popover
                ref="knowledgePopoverRef"
                placement="top-start"
                :width="280"
                trigger="click"
                popper-class="knowledge-popover"
                @show="isKnowledgePopoverVisible = true"
                @hide="isKnowledgePopoverVisible = false"
              >
                <template #default>
                  <div class="knowledge-list-container">
                    <div class="knowledge-list-header">
                      <span>选择知识库</span>
                      <button class="clear-btn" @click="clearKnowledgeSelection">取消选择</button>
                    </div>
                    <div class="knowledge-list">
                      <div
                        v-for="item in knowledgeList"
                        :key="item.id"
                        class="knowledge-item"
                        :class="{ 'is-selected': selectedKnowledgeId === item.id }"
                        @click="insertKnowledgeTag(item.id)"
                      >
                        <div class="item-name">
                          <el-icon>
                            <component :is="item.icon" />
                          </el-icon>
                          {{ item.name }}
                          <el-icon v-if="selectedKnowledgeId === item.id" class="item-check">
                            <Check />
                          </el-icon>
                        </div>
                      </div>
                    </div>
                  </div>
                </template>
                <template #reference>
                  <div class="knowledge-btn">
                    <el-icon class="knowledge-icon">
                      <DocumentCopy />
                    </el-icon>
                    <span class="knowledge-text">{{ selectedKnowledgeName }}</span>
                  </div>
                </template>
              </el-popover>

              <!-- 推理和联网按钮 -->
              <div class="feature-buttons">
                <div
                  class="feature-btn"
                  :class="{ active: isReasoningEnabled }"
                  @click="isReasoningEnabled = !isReasoningEnabled"
                >
                  <el-icon class="feature-icon">
                    <Operation />
                  </el-icon>
                  <span class="feature-text">推理</span>
                </div>
                <div
                  class="feature-btn"
                  :class="{ active: isWebSearchEnabled }"
                  @click="isWebSearchEnabled = !isWebSearchEnabled"
                >
                  <el-icon class="feature-icon">
                    <ChromeFilled />
                  </el-icon>
                  <span class="feature-text">联网</span>
                </div>

                <el-popover
                  ref="knowledgePopoverRef"
                  placement="top-start"
                  :width="280"
                  trigger="click"
                  popper-class="knowledge-popover"
                >
                  <template #default>
                    <div class="knowledge-list-container">
                      <div class="knowledge-list" @scroll="handleScroll">
                        <div
                          v-for="item in workflowList"
                          :key="item.id"
                          class="knowledge-item"
                          @click="chooseWorkflowItem(item)"
                          :class="{ 'is-selected': selectedWorkflowName === item.title }"
                        >
                          <div class="item-name">
                            {{ item.title }}
                          </div>
                        </div>
                        <!-- 加载提示 -->
                        <div v-if="isWorkflowLoading" class="loading-tip">加载中...</div>
                      </div>
                    </div>
                  </template>
                  <template #reference>
                    <div
                      class="feature-btn"
                      :class="{ active: isWorkflowVisible }"
                      @click="isWorkflowVisible = !isWorkflowVisible"
                    >
                      <el-icon class="feature-icon">
                        <SetUp />
                      </el-icon>
                      <span class="feature-text">{{ selectedWorkflowName }}</span>
                    </div>
                  </template>
                </el-popover>
              </div>
            </div>
          </template>
        </Sender>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
/* 编辑时去掉气泡样式 */
.user-bubble.editing {
  background: transparent !important;
  padding: 0;
}
:deep(.editing-bubble.el-bubble) {
  display: flex !important;
  width: 100% !important;
  justify-content: flex-start !important;
}

:deep(.editing-bubble .el-bubble__content) {
  flex: 1 !important;
  max-width: none !important;
  width: 100% !important;
}

/* 编辑卡片 */
.edit-card {
  width: 500px;
  box-sizing: border-box;
  border: 1px solid #dcdfe6;
  border-radius: 16px;
  padding: 12px;
  background: #ffffff;
  transition: all 0.2s ease;
}

/* textarea 样式 */
.edit-input :deep(.el-textarea__inner) {
  border: none !important;
  box-shadow: none !important;
  background: transparent !important;
  resize: none;
  padding: 0;
  font-size: 14px;
}

/* 按钮区域 */
.edit-actions {
  margin-top: 12px;
  display: flex;
  justify-content: flex-end;
  gap: 4px;
}

.copy-button-container {
  position: absolute;
  bottom: -28px;
  right: -10px;
  // opacity: 0;
  transform: translateY(10px);
  transition: all 0.3s ease;
  pointer-events: none;
  display: flex;
  justify-content: flex-end;

  .copy-btn {
    width: 24px;
    height: 24px;
    padding: 0;
    font-size: 16px;
    // background-color: #ffffff;
    cursor: pointer;
    pointer-events: auto;
    // float: right;
    border: none !important;
    color: #91949a;
    :deep(svg) {
      stroke-width: 3 !important;
    }

    &:hover {
      border-radius: 50%;
      transition: background-color 0.2s;
      background-color: #f1efef;
    }
  }
}
.chat-with-id-container {
  position: relative;
  display: flex;
  flex-direction: column;
  align-items: center;
  width: 100%;
  max-width: 800px;
  height: 100%;
  .chat-warp {
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    width: 100%;
    height: calc(100vh - 60px);

    // 默认图表样式
    .default-chart-wrapper {
      padding: 12px;
      border-bottom: 1px solid rgba(0, 0, 0, 0.08);
      background: linear-gradient(
        135deg,
        rgba(84, 112, 198, 0.02) 0%,
        rgba(84, 112, 198, 0.01) 100%
      );
    }

    .thinking-chain-warp {
      margin-bottom: 12px;
    }

    // Sender 输入框包装器
    .sender-wrapper {
      position: relative;
      width: 100%;

      // 新对话按钮 - 紧贴输入框左上角
      .new-chat-btn {
        position: absolute;
        top: -40px;
        left: 0;
        z-index: 10;
        display: inline-flex;
        gap: 6px;
        align-items: center;
        padding: 6px 12px;
        cursor: pointer;
        user-select: none;
        background-color: #ffffff;
        border: 1px solid rgb(0 0 0 / 10%);
        border-radius: 16px;
        box-shadow: 0 1px 2px rgb(0 0 0 / 5%);
        transition: all 0.2s ease;
        &:hover {
          background-color: rgb(0 87 255 / 4%);
          border-color: rgb(0 87 255 / 20%);
          box-shadow: 0 2px 4px rgb(0 87 255 / 10%);
          .btn-icon {
            color: #0057ff;
          }
        }
        .btn-icon {
          width: 16px;
          height: 16px;
          font-size: 16px;
          color: rgb(0 0 0 / 65%);
          transition: color 0.2s ease;
        }
        .btn-text {
          font-size: 13px;
          font-weight: 500;
          color: rgb(0 0 0 / 85%);
        }
      }
    }

    // 推理和联网按钮样式
    .feature-buttons {
      display: flex;
      gap: 8px;
      margin-left: 8px;
      .feature-btn {
        display: flex;
        gap: 4px;
        align-items: center;
        padding: 6px 12px;
        cursor: pointer;
        user-select: none;
        background-color: transparent;
        border: 1px solid rgb(0 0 0 / 10%);
        border-radius: 16px;
        transition: all 0.2s ease;
        &:hover {
          background-color: rgb(0 0 0 / 4%);
          border-color: rgb(0 0 0 / 15%);
        }
        &.active {
          background-color: rgb(0 87 255 / 8%);
          border-color: rgb(0 87 255 / 30%);
          .feature-icon {
            color: #0057ff;
          }
          .feature-text {
            color: #0057ff;
          }
        }
        .feature-icon {
          width: 16px;
          height: 16px;
          font-size: 16px;
          color: rgb(0 0 0 / 65%);
          transition: color 0.2s ease;
        }
        .feature-text {
          font-size: 13px;
          font-weight: 500;
          color: rgb(0 0 0 / 85%);
          transition: color 0.2s ease;
        }
      }
    }

    // 知识库按钮样式 (参考 Element Plus 风格)
    .knowledge-btn {
      display: flex;
      gap: 6px;
      align-items: center;
      justify-content: center;
      height: 32px;
      padding: 0 14px;
      cursor: pointer;
      user-select: none;
      background-color: #fff;
      border: 1px solid #dcdfe4;
      border-radius: 4px;
      font-size: 14px;
      font-weight: 500;
      color: #606266;
      transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
      position: relative;
      white-space: nowrap;
      overflow: hidden;
      text-overflow: ellipsis;

      &::before {
        content: "";
        position: absolute;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background: linear-gradient(
          135deg,
          transparent 0%,
          rgba(255, 255, 255, 0.2) 50%,
          transparent 100%
        );
        opacity: 0;
        transition: opacity 0.3s ease;
        pointer-events: none;
      }

      &:hover {
        color: #409eff;
        border-color: #c6e2ff;
        background-color: #f0f9ff;

        .knowledge-icon {
          color: #409eff;
        }

        &::before {
          opacity: 1;
        }
      }

      &:active {
        color: #0c5cff;
        border-color: #409eff;
        background-color: #e6f7ff;
        transform: scale(0.98);
      }

      .knowledge-icon {
        width: 16px;
        height: 16px;
        font-size: 16px;
        color: #909399;
        transition: color 0.3s ease;
        flex-shrink: 0;
      }

      .knowledge-text {
        font-size: 13px;
        font-weight: 500;
        color: inherit;
        transition: color 0.3s ease;
        max-width: 120px;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
      }
    }
  }
  :deep() {
    .el-bubble-list {
      padding-top: 24px;
    }
    .el-bubble {
      padding: 0 12px;
      padding-bottom: 24px;
    }
    .el-typewriter {
      overflow: hidden;
      border-radius: 12px;
    }
    .user-content {
      // 换行
      white-space: pre-wrap;
    }
    .markdown-body {
      background-color: transparent;
      width: auto;
      max-width: none;
      overflow: visible;
    }
    .markdown-elxLanguage-header-div {
      top: -25px !important;
    }

    // xmarkdown 样式
    .elx-xmarkdown-container {
      padding: 8px 4px;
      width: 100%;
      overflow: visible;
    }

    // 知识库 Popover 弹窗样式
    .knowledge-popover {
      box-shadow: 0 3px 12px rgba(0, 0, 0, 0.15);
      border-radius: 4px;
      padding: 0 !important;

      .el-popper__arrow {
        display: none;
      }

      [role="tooltip"] {
        padding: 0;
      }
    }
  }
  .chat-defaul-sender {
    width: 100%;
    margin-bottom: 22px;
  }
}

// 知识库列表容器
.knowledge-list-container {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

// 知识库列表标题
.knowledge-list-header {
  padding: 12px 16px 8px;
  font-size: 14px;
  font-weight: 600;
  color: #303133;
  border-bottom: 1px solid #ebeef5;
  display: flex;
  align-items: center;
  justify-content: space-between;

  span {
    flex: 1;
  }
}

// 知识库列表
.knowledge-list {
  display: flex;
  flex-direction: column;
  gap: 0;
  max-height: 300px;
  overflow-y: auto;

  // 知识库项目
  .knowledge-item {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 10px 12px;
    cursor: pointer;
    user-select: none;
    background-color: transparent;
    border-radius: 4px;
    transition: all 0.2s ease;
    position: relative;

    // 项目前的颜色指示器
    &::before {
      content: "";
      position: absolute;
      left: 0;
      top: 0;
      bottom: 0;
      width: 3px;
      border-radius: 0 2px 2px 0;
      background-color: var(--knowledge-color, #0057ff);
      opacity: 0;
      transition: opacity 0.2s ease;
    }

    &:hover {
      background-color: #f5f7fa;

      .item-name {
        color: #409eff;

        :deep(.el-icon) {
          color: #409eff;
        }
      }
    }

    &.is-selected {
      background-color: #f0f9ff;
      border-left: 3px solid;
      padding-left: 9px;

      &::before {
        opacity: 0;
      }

      .item-name {
        color: #0057ff;
        font-weight: 500;

        :deep(.el-icon) {
          color: #0057ff;
        }

        .item-check {
          color: #0057ff;
          font-size: 16px;
        }
      }
    }

    // 项目名称
    .item-name {
      display: flex;
      align-items: center;
      gap: 8px;
      flex: 1;
      font-size: 13px;
      color: #606266;
      transition: all 0.2s ease;
      width: 100%;

      :deep(.el-icon) {
        width: 16px;
        height: 16px;
        font-size: 16px;
        color: #909399;
        flex-shrink: 0;
        transition: color 0.2s ease;
      }

      .item-check {
        margin-left: auto;
        flex-shrink: 0;
      }
    }
  }
}

.loading-tip {
  text-align: center;
  padding: 12px;
  font-size: 14px;
  color: #909399;
}

// 清除按钮
.clear-btn {
  padding: 4px 12px;
  height: auto;
  background-color: transparent;
  border: none;
  border-radius: 4px;
  font-size: 12px;
  color: #909399;
  cursor: pointer;
  user-select: none;
  transition: all 0.2s ease;
  flex-shrink: 0;

  &:hover {
    color: #f56c6c;
    background-color: #fef0f0;
  }

  &:active {
    color: #c81d1d;
    background-color: #fde2e2;
  }
}
</style>
