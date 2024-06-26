<template>
  <div ref="editor" style="margin-bottom: 23px; border: 1px solid"></div>
  <v-btn @click="toggleConnection" class="mr-2">{{
    isConnected ? "Disconnect" : "Connect"
  }}</v-btn>
  <div>{{ firstChangeByLocalUser }}</div>
  <div>Version List</div>
  <v-list v-if="docVersions.length">
    <v-list-item
      v-for="(version, i) in docVersions"
      :key="version.id"
      :title="version.version_timestamp"
      :subtitle="`${i} of ${docVersions.length}`"
      @click="
        renderVersion(
          version.objectives,
          docVersions.length - 1 > i ? docVersions[i + 1].objectives : null
        )
      "
    />
  </v-list>
  <div v-else>No snapshot</div>
</template>

<script>
import { ref } from "vue";
import * as Y from "yjs";
import { WebsocketProvider } from "y-websocket";
import {
  ySyncPlugin,
  ySyncPluginKey,
  yCursorPlugin,
  yUndoPlugin,
  undo,
  redo,
} from "y-prosemirror";
import { EditorView } from "prosemirror-view";
import { EditorState, Plugin } from "prosemirror-state";
import { exampleSetup } from "prosemirror-example-setup";
// import { schema } from "prosemirror-schema-basic";
import { schema } from "./schema";
import { keymap } from "prosemirror-keymap";
import axios from "axios";
import { Base64 } from "js-base64";

export default {
  name: "Editor",
  setup() {
    const docId = "cXxJ1SHIQL87pbR21dcr5";
    const ydoc = new Y.Doc();
    ydoc.gc = false;
    const provider = new WebsocketProvider(
      "wss://logcarry-ywebsocket-3nuur.ondigitalocean.app/eln-note_objectives",
      docId,
      ydoc
    );
    const permanentUserData = new Y.PermanentUserData(ydoc);

    let view;

    return {
      docId,
      ydoc,
      provider,
      permanentUserData,
      view,
    };
  },
  data() {
    return {
      docVersions: [],
      firstChangeByLocalUser: false,
      isConnected: true,
    };
  },
  created() {
    this.getVersions();
  },
  mounted() {
    const yXmlFragment = this.ydoc.get("prosemirror", Y.XmlFragment);
    const colors = [
      { light: "#ecd44433", dark: "#ecd444" },
      { light: "#ee635233", dark: "#ee6352" },
      { light: "#6eeb8333", dark: "#6eeb83" },
    ];
    const state = EditorState.create({
      schema,
      plugins: [
        ySyncPlugin(yXmlFragment, {
          permanentUserData: this.permanentUserData,
          colors,
        }),
        yCursorPlugin(this.provider.awareness),
        yUndoPlugin(),
        keymap({
          "Mod-z": undo,
          "Mod-y": redo,
          "Mod-Shift-z": redo,
        }),
        new Plugin({
          filterTransaction: (transaction, editorState) => {
            console.log("[Plugin]-filterTransaction runs");
            console.log({ transaction });
            if (Object.keys(transaction.meta).length === 0) {
              if (!this.firstChangeByLocalUser && transaction.updated !== 0) {
                console.log("firstChangeByLocalUser");
                this.firstChangeByLocalUser = true;
                this.permanentUserData.setUserMapping(
                  this.ydoc,
                  this.ydoc.clientID,
                  "Alice"
                );
              }
            }
            return true;
          },
        }),
      ].concat(exampleSetup({ schema })),
    });
    this.view = new EditorView(this.$refs.editor, {
      state: state,
    });
  },
  // please adjust this and ensure it is disconnected,
  // else the auto-versioning won't work, and the server might run out of connection
  beforeUnmount() {
    this.provider.disconnect();
  },
  methods: {
    getVersions() {
      axios
        .get(
          "https://logcarry-ywebsocket-3nuur.ondigitalocean.app/api/docs-versions/eln-note_objectives/" +
            this.docId
        )
        .then((response) => {
          this.docVersions = response.data;
          console.log(response.data);
        })
        .catch((error) => {
          console.log(error);
        });
    },
    toggleConnection() {
      if (this.isConnected) {
        this.provider.disconnect();
        console.log("disconnected");
      } else {
        this.provider.connect();
        this.getVersions();
        console.log("connected");
      }
      this.isConnected = !this.isConnected;
    },
    renderVersion(currSnapshot, prevSnapshot) {
      console.log("currSnapshot:", currSnapshot);
      console.log("prevSnapshot:", prevSnapshot);

      // Check if currSnapshot and prevSnapshot are defined
      if (currSnapshot != null) {
        let convertedCurrSnapshot = Base64.toUint8Array(currSnapshot);
        let convertedPrevSnapshot =
          prevSnapshot != null ? Base64.toUint8Array(prevSnapshot) : null;

        this.view.dispatch(
          this.view.state.tr.setMeta(ySyncPluginKey, {
            snapshot: Y.decodeSnapshot(convertedCurrSnapshot),
            prevSnapshot:
              convertedPrevSnapshot != null
                ? Y.decodeSnapshot(convertedPrevSnapshot)
                : Y.emptySnapshot,
          })
        );
      }
    },
  },
};
</script>

<style>
.ProseMirror {
  position: relative;
}

.ProseMirror {
  word-wrap: break-word;
  white-space: pre-wrap;
  -webkit-font-variant-ligatures: none;
  font-variant-ligatures: none;
}

.ProseMirror pre {
  white-space: pre-wrap;
}

.ProseMirror li {
  position: relative;
}

.ProseMirror-hideselection *::selection {
  background: transparent;
}
.ProseMirror-hideselection *::-moz-selection {
  background: transparent;
}
.ProseMirror-hideselection {
  caret-color: transparent;
}

.ProseMirror-selectednode {
  outline: 2px solid #8cf;
}

/* Make sure li selections wrap around markers */

li.ProseMirror-selectednode {
  outline: none;
}

li.ProseMirror-selectednode:after {
  content: "";
  position: absolute;
  left: -32px;
  right: -2px;
  top: -2px;
  bottom: -2px;
  border: 2px solid #8cf;
  pointer-events: none;
}
.ProseMirror-textblock-dropdown {
  min-width: 3em;
}

.ProseMirror-menu {
  margin: 0 -4px;
  line-height: 1;
}

.ProseMirror-tooltip .ProseMirror-menu {
  width: -webkit-fit-content;
  width: fit-content;
  white-space: pre;
}

.ProseMirror-menuitem {
  margin-right: 3px;
  display: inline-block;
}

.ProseMirror-menuseparator {
  border-right: 1px solid #ddd;
  margin-right: 3px;
}

.ProseMirror-menu-dropdown,
.ProseMirror-menu-dropdown-menu {
  font-size: 90%;
  white-space: nowrap;
}

.ProseMirror-menu-dropdown {
  vertical-align: 1px;
  cursor: pointer;
  position: relative;
  padding-right: 15px;
}

.ProseMirror-menu-dropdown-wrap {
  padding: 1px 0 1px 4px;
  display: inline-block;
  position: relative;
}

.ProseMirror-menu-dropdown:after {
  content: "";
  border-left: 4px solid transparent;
  border-right: 4px solid transparent;
  border-top: 4px solid currentColor;
  opacity: 0.6;
  position: absolute;
  right: 4px;
  top: calc(50% - 2px);
}

.ProseMirror-menu-dropdown-menu,
.ProseMirror-menu-submenu {
  position: absolute;
  background: white;
  color: #666;
  border: 1px solid #aaa;
  padding: 2px;
}

.ProseMirror-menu-dropdown-menu {
  z-index: 15;
  min-width: 6em;
}

.ProseMirror-menu-dropdown-item {
  cursor: pointer;
  padding: 2px 8px 2px 4px;
}

.ProseMirror-menu-dropdown-item:hover {
  background: #f2f2f2;
}

.ProseMirror-menu-submenu-wrap {
  position: relative;
  margin-right: -4px;
}

.ProseMirror-menu-submenu-label:after {
  content: "";
  border-top: 4px solid transparent;
  border-bottom: 4px solid transparent;
  border-left: 4px solid currentColor;
  opacity: 0.6;
  position: absolute;
  right: 4px;
  top: calc(50% - 4px);
}

.ProseMirror-menu-submenu {
  display: none;
  min-width: 4em;
  left: 100%;
  top: -3px;
}

.ProseMirror-menu-active {
  background: #eee;
  border-radius: 4px;
}

.ProseMirror-menu-active {
  background: #eee;
  border-radius: 4px;
}

.ProseMirror-menu-disabled {
  opacity: 0.3;
}

.ProseMirror-menu-submenu-wrap:hover .ProseMirror-menu-submenu,
.ProseMirror-menu-submenu-wrap-active .ProseMirror-menu-submenu {
  display: block;
}

.ProseMirror-menubar {
  border-top-left-radius: inherit;
  border-top-right-radius: inherit;
  position: relative;
  min-height: 1em;
  color: #666;
  padding: 1px 6px;
  top: 0;
  left: 0;
  right: 0;
  border-bottom: 1px solid silver;
  background: white;
  z-index: 10;
  -moz-box-sizing: border-box;
  box-sizing: border-box;
  overflow: visible;
}

.ProseMirror-icon {
  display: inline-block;
  line-height: 0.8;
  vertical-align: -2px; /* Compensate for padding */
  padding: 2px 8px;
  cursor: pointer;
}

.ProseMirror-menu-disabled.ProseMirror-icon {
  cursor: default;
}

.ProseMirror-icon svg {
  fill: currentColor;
  height: 1em;
}

.ProseMirror-icon span {
  vertical-align: text-top;
}
.ProseMirror-gapcursor {
  display: none;
  pointer-events: none;
  position: absolute;
}

.ProseMirror-gapcursor:after {
  content: "";
  display: block;
  position: absolute;
  top: -2px;
  width: 20px;
  border-top: 1px solid black;
  animation: ProseMirror-cursor-blink 1.1s steps(2, start) infinite;
}

@keyframes ProseMirror-cursor-blink {
  to {
    visibility: hidden;
  }
}

.ProseMirror-focused .ProseMirror-gapcursor {
  display: block;
}
/* Add space around the hr to make clicking it easier */

.ProseMirror-example-setup-style hr {
  padding: 2px 10px;
  border: none;
  margin: 1em 0;
}

.ProseMirror-example-setup-style hr:after {
  content: "";
  display: block;
  height: 1px;
  background-color: silver;
  line-height: 2px;
}

.ProseMirror ul,
.ProseMirror ol {
  padding-left: 30px;
}

.ProseMirror blockquote {
  padding-left: 1em;
  border-left: 3px solid #eee;
  margin-left: 0;
  margin-right: 0;
}

.ProseMirror-example-setup-style img {
  cursor: default;
}

.ProseMirror-prompt {
  background: white;
  padding: 5px 10px 5px 15px;
  border: 1px solid silver;
  position: fixed;
  border-radius: 3px;
  z-index: 11;
  box-shadow: -0.5px 2px 5px rgba(0, 0, 0, 0.2);
}

.ProseMirror-prompt h5 {
  margin: 0;
  font-weight: normal;
  font-size: 100%;
  color: #444;
}

.ProseMirror-prompt input[type="text"],
.ProseMirror-prompt textarea {
  background: #eee;
  border: none;
  outline: none;
}

.ProseMirror-prompt input[type="text"] {
  padding: 0 4px;
}

.ProseMirror-prompt-close {
  position: absolute;
  left: 2px;
  top: 1px;
  color: #666;
  border: none;
  background: transparent;
  padding: 0;
}

.ProseMirror-prompt-close:after {
  content: "âœ•";
  font-size: 12px;
}

.ProseMirror-invalid {
  background: #ffc;
  border: 1px solid #cc7;
  border-radius: 4px;
  padding: 5px 10px;
  position: absolute;
  min-width: 10em;
}

.ProseMirror-prompt-buttons {
  margin-top: 5px;
  display: none;
}
#editor,
.editor {
  background: white;
  color: black;
  background-clip: padding-box;
  border-radius: 4px;
  border: 2px solid rgba(0, 0, 0, 0.2);
  padding: 5px 0;
  margin-bottom: 23px;
}

.ProseMirror p:first-child,
.ProseMirror h1:first-child,
.ProseMirror h2:first-child,
.ProseMirror h3:first-child,
.ProseMirror h4:first-child,
.ProseMirror h5:first-child,
.ProseMirror h6:first-child {
  margin-top: 10px;
}

.ProseMirror {
  padding: 4px 8px 4px 14px;
  line-height: 1.2;
  outline: none;
}

.ProseMirror p {
  margin-bottom: 1em;
}
</style>
