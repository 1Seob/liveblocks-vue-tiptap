<script setup>
import { ref, onBeforeUnmount } from 'vue';
import { createClient, LiveObject } from '@liveblocks/client';
import { getYjsProviderForRoom } from '@liveblocks/yjs';
import { Editor, EditorContent } from '@tiptap/vue-3';
import StarterKit from '@tiptap/starter-kit';
import Collaboration from '@tiptap/extension-collaboration';
import CollaborationCursor from '@tiptap/extension-collaboration-cursor';

import Underline from '@tiptap/extension-underline';
import TextAlign from '@tiptap/extension-text-align';
import TextStyle from '@tiptap/extension-text-style';
import Color from '@tiptap/extension-color';
import Link from '@tiptap/extension-link';
import TaskList from '@tiptap/extension-task-list';
import TaskItem from '@tiptap/extension-task-item';

const roomId = ref('');
const accessToken = ref('');
const displayName = ref('User ' + Math.floor(Math.random() * 1000));
const joined = ref(false);
const status = ref('Not connected');

let client = null;
let room = null;
let yProvider = null;
let ydoc = null;
let editor = null;
let leaveFn = null; // enterRoom이 반환하는 leave 함수

// 저장 상태 표시용
const saving = ref(false);
const savedAt = ref('');

async function joinRoom() {
  try {
    if (!roomId.value || !accessToken.value) {
      alert('roomId와 accessToken을 입력하세요.');
      return;
    }

    client = createClient({
      authEndpoint: async () => ({ token: accessToken.value }),
    });

    // Storage 루트가 비어 있을 수 있으므로 initialStorage 제공
    const { room: r, leave: lv } = client.enterRoom(roomId.value, {
      initialStorage: new LiveObject({}),
    });
    room = r;
    leaveFn = lv;

    yProvider = getYjsProviderForRoom(room);
    ydoc = yProvider.getYDoc();

    status.value = 'connecting';
    yProvider.on('sync', (synced) => {
      status.value = synced ? 'connected' : 'connecting';
    });
    yProvider.on('error', (err) => {
      console.error('[yProvider error]', err);
      alert('Provider error: ' + (err?.message || err));
    });

    const userColor = stringToColor(displayName.value);

    editor = new Editor({
      extensions: [
        StarterKit.configure({ history: false }),
        // Yjs 문서의 "tiptap" 필드를 사용
        Collaboration.configure({ document: ydoc, field: 'tiptap' }),

        // ✨ 커서 커스터마이징
        CollaborationCursor.configure({
          provider: yProvider,
          user: { name: displayName.value, color: userColor },
          render: (user) => {
            const root = document.createElement('span');
            root.className = 'lb-cursor';
            root.style.borderColor = user.color;

            const dot = document.createElement('span');
            dot.className = 'lb-cursor__dot';
            dot.style.backgroundColor = user.color;

            const label = document.createElement('span');
            label.className = 'lb-cursor__label';
            label.textContent = user.name || 'anonymous';
            label.style.backgroundColor = user.color;
            label.style.color = getReadableTextColor(user.color);

            root.appendChild(dot);
            root.appendChild(label);
            return root;
          },
        }),

        Underline,
        TextStyle,
        Color,
        TextAlign.configure({ types: ['heading', 'paragraph'] }),
        Link.configure({
          autolink: true,
          openOnClick: true,
          linkOnPaste: true,
          HTMLAttributes: { rel: 'noopener noreferrer', target: '_blank' },
        }),
        TaskList,
        TaskItem.configure({ nested: true }),
      ],
      content: `
        <h1>Liveblocks + Tiptap (Vue) 테스트</h1>
        <p>같은 roomId + 유효한 토큰으로 다른 브라우저에서 접속해보세요.</p>
      `,
    });

    joined.value = true;
  } catch (e) {
    console.error('[joinRoom] failed:', e);
    alert('Join 중 오류: ' + (e?.message || e));
  }
}

function leaveRoom() {
  try {
    if (editor) {
      editor.destroy();
      editor = null;
    }
    if (yProvider) {
      yProvider.destroy?.();
      yProvider = null;
    }
    // enterRoom이 반환한 leave() 사용
    if (leaveFn) {
      leaveFn();
      leaveFn = null;
    }
    room = null;
    ydoc = null;
    client = null;
    joined.value = false;
    status.value = 'Not connected';
  } catch (e) {
    console.error('[leaveRoom] failed:', e);
  }
}

onBeforeUnmount(() => leaveRoom());

/**
 * 현재 Tiptap 문서를 Liveblocks Storage에 저장
 * - root LiveObject의 "lastSavedDoc" 키에 JSON 저장
 * - 서버: GET /v2/rooms/{roomId}/storage?format=json 으로 확인
 */
async function saveToStorage() {
  try {
    if (!joined.value || !room || !editor) {
      alert('먼저 룸에 접속하세요.');
      return;
    }
    saving.value = true;
    const docJSON = editor.getJSON();

    // Storage 루트 가져오기
    const { root } = await room.getStorage();

    // 하나의 트랜잭션으로 묶어서 기록 (선택이지만 권장)
    room.batch(() => {
      root.set('doc', docJSON);
    });

    savedAt.value = new Date().toLocaleString();
  } catch (e) {
    console.error('[saveToStorage] failed:', e);
    alert('Storage 저장 중 오류: ' + (e?.message || e));
  } finally {
    saving.value = false;
  }
}

// === Utils ===
function stringToColor(s) {
  let h = 0;
  for (let i = 0; i < s.length; i++) h = s.charCodeAt(i) + ((h << 5) - h);
  const c = (h & 0x00ffffff).toString(16).toUpperCase();
  return '#' + '00000'.substring(0, 6 - c.length) + c;
}

function getReadableTextColor(hex) {
  const { r, g, b } = hexToRgb(hex);
  const luma = 0.2126 * r + 0.7152 * g + 0.0722 * b;
  return luma > 160 ? '#111' : '#fff';
}
function hexToRgb(hex) {
  const h = hex.replace('#', '');
  const bigint = parseInt(h, 16);
  return { r: (bigint >> 16) & 255, g: (bigint >> 8) & 255, b: bigint & 255 };
}
</script>

<template>
  <div class="wrap">
    <h2>Liveblocks × Tiptap (Vue) — Access Token</h2>

    <div v-if="!joined" class="row">
      <input
        v-model="roomId"
        placeholder="roomId (예: group:1:meeting:1)"
        style="flex: 1 1 240px"
      />
      <input
        v-model="accessToken"
        placeholder="accessToken (JWT)"
        style="flex: 3 1 380px"
      />
      <input
        v-model="displayName"
        placeholder="표시 이름"
        style="flex: 0 1 180px"
      />
      <button @click="joinRoom" class="btn primary">Join</button>
    </div>

    <div v-else class="row">
      <span class="badge"
        >Room: <b>{{ roomId }}</b></span
      >
      <span class="badge"
        >Me: <b>{{ displayName }}</b></span
      >
      <button @click="leaveRoom" class="btn danger">Leave</button>
    </div>

    <div v-if="joined">
      <div class="toolbar">
        <button
          class="btn"
          :class="{ active: editor?.isActive('bold') }"
          @click="editor?.chain().focus().toggleBold().run()"
        >
          B
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive('italic') }"
          @click="editor?.chain().focus().toggleItalic().run()"
        >
          <i>I</i>
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive('underline') }"
          @click="editor?.chain().focus().toggleUnderline().run()"
        >
          <u>U</u>
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive('strike') }"
          @click="editor?.chain().focus().toggleStrike().run()"
        >
          <s>S</s>
        </button>

        <input
          class="color"
          type="color"
          title="Text color"
          @input="(e) => editor?.chain().focus().setColor(e.target.value).run()"
        />

        <span class="sep"></span>

        <button
          class="btn"
          :class="{ active: editor?.isActive('paragraph') }"
          @click="editor?.chain().focus().setParagraph().run()"
        >
          본문
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive('heading', { level: 1 }) }"
          @click="editor?.chain().focus().toggleHeading({ level: 1 }).run()"
        >
          H1
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive('heading', { level: 2 }) }"
          @click="editor?.chain().focus().toggleHeading({ level: 2 }).run()"
        >
          H2
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive('heading', { level: 3 }) }"
          @click="editor?.chain().focus().toggleHeading({ level: 3 }).run()"
        >
          H3
        </button>

        <span class="sep"></span>

        <button
          class="btn"
          :class="{ active: editor?.isActive({ textAlign: 'left' }) }"
          @click="editor?.chain().focus().setTextAlign('left').run()"
        >
          좌
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive({ textAlign: 'center' }) }"
          @click="editor?.chain().focus().setTextAlign('center').run()"
        >
          중
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive({ textAlign: 'right' }) }"
          @click="editor?.chain().focus().setTextAlign('right').run()"
        >
          우
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive({ textAlign: 'justify' }) }"
          @click="editor?.chain().focus().setTextAlign('justify').run()"
        >
          양쪽
        </button>

        <span class="sep"></span>

        <button
          class="btn"
          :class="{ active: editor?.isActive('bulletList') }"
          @click="editor?.chain().focus().toggleBulletList().run()"
        >
          • 목록
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive('orderedList') }"
          @click="editor?.chain().focus().toggleOrderedList().run()"
        >
          1. 목록
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive('taskList') }"
          @click="editor?.chain().focus().toggleTaskList().run()"
        >
          ☑ 체크리스트
        </button>
        <button
          class="btn"
          :class="{ active: editor?.isActive('blockquote') }"
          @click="editor?.chain().focus().toggleBlockquote().run()"
        >
          ❝ 인용
        </button>

        <button
          class="btn"
          :class="{ active: editor?.isActive('codeBlock') }"
          @click="editor?.chain().focus().toggleCodeBlock().run()"
        >
          &lt;/&gt; 코드
        </button>

        <button
          class="btn"
          @click="editor?.chain().focus().setHorizontalRule().run()"
        >
          ― 구분선
        </button>

        <span class="sep"></span>

        <button
          class="btn"
          :class="{ active: editor?.isActive('link') }"
          @click="setLink()"
        >
          🔗 링크
        </button>
        <button class="btn" @click="editor?.chain().focus().unsetLink().run()">
          ❌ 링크해제
        </button>

        <span class="sep"></span>

        <!-- 💾 Storage에 현재 문서 JSON 저장 -->
        <button class="btn" :disabled="saving" @click="saveToStorage">
          💾 Storage에 저장
        </button>
      </div>

      <div
        class="editor"
        :style="{
          outline:
            status === 'connected' ? '2px solid #22c55e' : '2px dashed #94a3b8',
        }"
      >
        <EditorContent :editor="editor" />
      </div>
      <div class="status">Status: {{ status }}</div>
      <div class="status" v-if="savedAt">Last saved: {{ savedAt }}</div>
    </div>
  </div>
</template>

<script>
export default {
  methods: {
    setLink() {
      const prev = this.editor?.getAttributes('link').href || '';
      const url = window.prompt('링크 URL을 입력하세요', prev);
      if (url === null) return;
      if (url === '') {
        this.editor?.chain().focus().unsetLink().run();
        return;
      }
      this.editor
        ?.chain()
        .focus()
        .extendMarkRange('link')
        .setLink({ href: url })
        .run();
    },
  },
  computed: {
    // setup()의 editor를 template에서 this.editor로 접근하기 위한 우회
    editor() {
      return this.$?.setupState?.editor ?? null;
    },
  },
};
</script>

<style>
/* Layout */
.wrap {
  max-width: 920px;
  margin: 24px auto;
  padding: 0 16px;
}
.row {
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  margin-bottom: 12px;
}
input {
  padding: 8px 10px;
  border: 1px solid #ddd;
  border-radius: 8px;
}
.btn {
  padding: 6px 10px;
  border: 1px solid #e5e7eb;
  background: #fff;
  border-radius: 8px;
  cursor: pointer;
}
.btn[disabled] {
  opacity: 0.6;
  cursor: not-allowed;
}
.btn.primary {
  background: #0ea5e9;
  color: #fff;
  border-color: #0ea5e9;
}
.btn.danger {
  background: #ef4444;
  color: #fff;
  border-color: #ef4444;
}
.btn.active {
  background: #e5f2ff;
  border-color: #60a5fa;
}
.toolbar {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  padding: 8px;
  border: 1px solid #e5e7eb;
  border-radius: 12px;
  margin: 12px 0;
  background: #fafafa;
}
.sep {
  width: 1px;
  height: 24px;
  background: #e5e7eb;
  margin: 0 4px;
}
.color {
  width: 36px;
  height: 32px;
  padding: 0;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  background: #fff;
}
.editor {
  border: 1px solid #ddd;
  border-radius: 12px;
  padding: 14px;
  min-height: 300px;
  background: #fff;
}
.status {
  font-size: 12px;
  color: #666;
  margin-top: 6px;
}
.badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 8px;
  border-radius: 999px;
  background: #f1f5f9;
}

/* ✨ 커스텀 커서 스타일 (Quill-cursors 느낌) */
@keyframes lb-blink {
  0%,
  49% {
    opacity: 1;
  }
  50%,
  100% {
    opacity: 0.15;
  }
}
.lb-cursor {
  position: relative;
  border-left: 3px solid;
  margin-left: -1px;
  pointer-events: none;
  animation: lb-blink 1.2s step-start infinite;
}
.lb-cursor__dot {
  position: absolute;
  width: 8px;
  height: 8px;
  border-radius: 999px;
  left: -5px;
  top: -5px;
}
.lb-cursor__label {
  position: absolute;
  transform: translateY(-100%);
  left: -2px;
  top: -6px;
  padding: 2px 8px;
  font-size: 12px;
  font-weight: 700;
  border-radius: 8px;
  color: #fff;
  white-space: nowrap;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.18);
  user-select: none;
}
</style>
