<template>
  <!-- Token lists cannot be inline -->
  <div class="form-control">
    <label v-if="label" v-bind:for="fieldID" v-html="label"></label>
    <!-- Else: Normal input w/o reset button -->
    <div class="token-list" v-on:click="$refs.input.focus()">
      <span
        v-for="(token, idx) in value"
        v-bind:key="idx"
        class="token"
        v-on:click="removeToken(idx)"
      >
        {{ token }}
      </span>
      <input
        v-bind:id="fieldID"
        ref="input"
        v-model="inputValue"
        class="inline"
        type="text"
        v-on:keypress="handleKey"
      >
    </div>
  </div>
</template>

<script>
export default {
  name: 'TokenList',
  props: {
    value: {
      type: Array,
      default: function () { return [] }
    },
    label: {
      type: String,
      default: ''
    },
    name: {
      type: String,
      default: ''
    }
  },
  data: function () {
    return {
      inputValue: ''
    }
  },
  computed: {
    fieldID: function () {
      return 'field-input-' + this.name
    }
  },
  methods: {
    handleKey: function (event) {
      if (this.inputValue.trim() === '') {
        return
      }

      if ([ 'Space', 'Enter', 'Comma', 'Tab' ].includes(event.code)) {
        const arr = this.value.map(token => token)
        // Don't add duplicates
        if (!arr.includes(this.inputValue.trim())) {
          arr.push(this.inputValue.trim())
          this.$emit('input', arr)
        }
        this.inputValue = ''
      }
    },
    removeToken: function (idx) {
      const arr = this.value.map(token => token)
      arr.splice(idx, 1)
      this.$emit('input', arr)
    }
  }
}
</script>

<style lang="less">
body {
  div.token-list {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
    font-size: 12px;
    padding: 6px;

    .token {
      display: inline-block;
      border-radius: 4px;
      margin: 2px;
      padding: 2px;
    }

    input {
      background-color: transparent;
      font-family: inherit;
      color: inherit;
      border: none;
    }
  }
  &.dark {
    div.token-list {
      .token {
        background-color: rgb(70, 70, 70);
        &:hover {
          background-color: rgb(110, 30, 30);
        }
      }

      input {
        background-color: transparent;
        font-family: inherit;
        color: inherit;
        border: none;
      }
    }
  }
}
</style>
