<template>
  <div
    id="sidebar"
    v-bind:class="{ 'open': showSidebar }"
  >
    <div id="sidebar-tabs" role="tablist">
      <div
        role="tab"
        v-bind:aria-label="attachmentsLabel"
        data-target="sidebar-files"
        v-bind:class="{
          'sidebar-tab': true,
          'active': currentTab === 'attachments'
        }"
        v-bind:title="attachmentsLabel"
        v-on:click="currentTab = 'attachments'"
      >
        <clr-icon shape="attachment" role="presentation"></clr-icon>
      </div>
      <div
        role="tab"
        v-bind:aria-label="referencesLabel"
        data-target="sidebar-bibliography"
        v-bind:class="{
          'sidebar-tab': true,
          'active': currentTab === 'references'
        }"
        v-bind:title="referencesLabel"
        v-on:click="currentTab = 'references'"
      >
        <clr-icon shape="book" role="presentation"></clr-icon>
      </div>
      <div
        role="tab"
        v-bind:aria-label="tocLabel"
        data-target="sidebar-toc"
        v-bind:class="{
          'sidebar-tab': true,
          'active': currentTab === 'toc'
        }"
        v-bind:title="tocLabel"
        v-on:click="currentTab = 'toc'"
      >
        <clr-icon shape="indented-view-list" role="presentation"></clr-icon>
      </div>
    </div>

    <!-- Now the tab containers -->
    <div v-show="currentTab === 'attachments'">
      <!-- Other files contents -->
      <h1>
        {{ attachmentsLabel }}
        <clr-icon
          id="open-dir-external"
          v-bind:title="openDirLabel"
          shape="folder"
          class="is-solid"
        ></clr-icon>
      </h1>

      <!-- Render all attachments -->
      <p v-if="attachments.length === 0">
        {{ noAttachmentsMessage }}
      </p>
      <template v-else>
        <a
          v-for="(attachment, idx) in attachments"
          v-bind:key="idx"
          class="attachment"
          draggable="true"
          v-bind:data-link="attachment.path"
          v-bind:data-hash="attachment.hash"
          v-bind:title="attachment.path"
          v-bind:href="`safe-file://${attachment.path}`"
          v-on:dragstart="handleDragStart($event, attachment.path)"
        >
          <span v-html="getIcon(attachment.path)"></span>
          {{ attachment.name }}
        </a>
      </template>
    </div>
    <div v-show="currentTab === 'references'">
      <!-- References -->
      <h1>{{ referencesLabel }}</h1>
      <p v-if="bibContents === undefined">
        No references
      </p>
      <template v-else>
        <div v-html="bibContents[0].bibstart"></div>
        <div v-for="(entry, idx) of bibContents[1]" v-bind:key="idx">
          <!-- TODO: Doesn't work right now because the bibliography is never rendered as of now -->
          <div v-html="entry"></div>
        </div>
        <div v-html="bibContents[0].bibend"></div>
      </template>
    </div>
    <div v-show="currentTab === 'toc'">
      <!-- Table of Contents -->
      <h1>{{ tocLabel }}</h1>
      <!-- Show the ToC entries -->
      <div
        v-for="(entry, idx) of tableOfContents"
        v-bind:key="idx"
        class="toc-entry-container"
        v-bind:style="{
          'margin-left': `${entry.level * 10}px`
        }"
        v-on:click="$root.$emit('toc-line', entry.line)"
      >
        <div class="toc-level">
          {{ entry.renderedLevel }}
        </div>
        <div class="toc-entry" v-bind:data-line="entry.line" v-html="entry.text"></div>
      </div>
    </div>
  </div>
</template>

<script>
import { trans } from '../common/i18n'
import { ClarityIcons } from '@clr/icons'
import path from 'path'
import { ipcRenderer } from 'electron'

const FILETYPES_IMG = [
  '.jpg',
  '.jpeg',
  '.svg',
  '.gif',
  '.png',
  '.tiff',
  '.tif'
]

export default {
  name: 'Sidebar',
  props: {
    showSidebar: {
      type: Boolean,
      default: true
    }
  },
  data: function () {
    return {
      currentTab: 'attachments',
      bibContents: undefined
    }
  },
  computed: {
    attachmentsLabel: function () {
      return trans('gui.attachments')
    },
    referencesLabel: function () {
      return trans('gui.citeproc.references_heading')
    },
    tocLabel: function () {
      return trans('gui.table_of_contents')
    },
    openDirLabel: function () {
      return trans('gui.attachments_open_dir')
    },
    noAttachmentsMessage: function () {
      return trans('gui.no_attachments')
    },
    attachments: function () {
      const currentDir = this.$store.state.selectedDirectory
      if (currentDir === null) {
        return []
      } else {
        return currentDir.attachments
      }
    },
    tableOfContents: function () {
      return this.$store.state.tableOfContents
    }
  },
  mounted: function () {
    ipcRenderer.on('citeproc-renderer', (event, { command, payload }) => {
      if (command === 'citeproc-bibliography') {
        this.bibContents = payload
      }
    })
  },
  methods: {
    getIcon: function (attachmentPath) {
      const fileExtIcon = ClarityIcons.get('file-ext')
      if (typeof fileExtIcon === 'string') {
        return fileExtIcon.replace('EXT', path.extname(attachmentPath).slice(1, 4))
      } else {
        return ''
      }
    },
    handleDragStart: function (event, attachmentPath) {
      // When dragging files from here onto the editor instance, users want
      // to have the appropriate link placed automatically, that is: images
      // should be wrapped in appropriate image tags, whereas documents
      // should be linked to enable click & open. We have to do this on
      // this end, because when trying to override data during drop it
      // won't work.
      const ext = path.extname(attachmentPath).toLowerCase()
      const basename = path.basename(attachmentPath)
      const uri = decodeURIComponent(attachmentPath)
      const data = FILETYPES_IMG.includes(ext) ? `![${basename}](${uri})` : `[${basename}](${uri})`
      event.dataTransfer.setData('text', data)
    }
  }
}
</script>

<style lang="less">
body {
  @button-margin: 5px;
  @border-radius: 5px;
  @button-size: 5px;
  @button-icon-size: 5px;
  #sidebar {
    background-color: rgba(30, 30, 30, .3);

    position: fixed;
    z-index: 2; // Otherwise the editor will have a higher z-index somehow
    top: 0;
    right: -20%;
    bottom: 0;
    width: 20%;
    overflow-y: auto;
    overflow-x: hidden;
    transition: right 0.5s ease 0s;

    &.open {
      right: 0%;
    }

    // Enable tabs
    #sidebar-tabs {
      display: flex;
      justify-content: space-evenly;

      .sidebar-tab {
        flex-grow: 1;
        text-align: center;
      }
    }

    #open-dir-external {
        padding: @button-margin;
        border-radius: @border-radius;
        display: inline-block;
        width: @button-size;
        height: @button-size;

        clr-icon {
            width: @button-icon-size;
            height: @button-icon-size;
        }
    }

    h1 {
        padding: 10px;
        font-size: 16px;
    }

    p { padding: 10px; }

    a.attachment {
        display: block;
        margin: 10px;
        padding: 4px;
        text-decoration: none;
        color: inherit;
        // Padding 4px + 4px margin + 24px icon width = 32px
        text-indent: -32px;
        padding-left: 32px;
        svg {
          width: 24px;
          height: 24px;
          margin-right: 4px;
          vertical-align: bottom;
          margin-bottom: -1px;
          // Necessary to give the extension icons the correct colour
          fill: currentColor;
        }
    }

    // Bibliography entries
    div.csl-bib-body {
      div.csl-entry {
        display: list-item;
        list-style-type: square;
        margin: 1em 0.2em 1em 1.8em;
        font-size: 80%;
        user-select: text;
        cursor: text;
      }
    }

    // Table of Contents entries
    div.toc-entry-container {
      // Clever calculation based on the data-level property
      // margin-left: calc(attr(data-level) * 10px);
      display: flex;

      div.toc-level { flex-shrink: 1; }
      div.toc-entry { flex-grow: 3; }
    }
  }

  &.dark {
    #sidebar {
      background-color: rgba(0, 0, 0, 0.8);
      color: rgb(230, 230, 230);
    }
  }
}

body.darwin div#sidebar {
  top: 40px; // On macOS the documents titlebar is 30px high, so we want to offset the sidebar by that.
}
</style>
