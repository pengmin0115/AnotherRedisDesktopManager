<template>
  <div class="text-formated-container">
    <p :title="fullCommand" class="command-preview">{{previewCommand}}</p>

    <div class="collapse-container">
      <el-button class="collapse-btn" type="text" @click="toggleCollapse">{{
          $t('message.' + collapseText)
        }}
      </el-button>
    </div>

    <JsonViewer
      v-if='show'
      :value="newContent"
      :expand-depth="previousDeep">
    </JsonViewer>
  </div>
</template>

<script type="text/javascript">
import storage from '@/storage';
import shell from 'child_process';
import JsonViewer from 'vue-json-viewer';

export default {
  data() {
    return {
      show: true,
      previousDeep: 3,
      collapseText: 'collapse_all',
      execResult: '',
      fullCommand: '',
      previewCommand: '',
      previewContentMax: 300,
    };
  },
  components: { JsonViewer },
  props: ['content', 'name', 'dataMap', 'redisKey'],
  computed: {
    newContent() {
      if (this.$util.isJson(this.execResult)) {
        return JSON.parse(this.execResult);
      }

      return this.execResult;
    },
  },
  watch: {
    content() {
      this.execCommand();
    },
  },
  methods: {
    toggleCollapse() {
      this.previousDeep = this.previousDeep ? 0 : Infinity;
      this.collapseText = this.previousDeep ? 'collapse_all' : 'expand_all';

      // reload json tree
      this.show = false;
      this.$nextTick(() => {
        this.show = true;
      });
    },
    getCommand() {
      const formatter = storage.getCustomFormatter(this.name);

      if (!formatter) {
        return false;
      }

      const command = formatter.command;
      const params = formatter.params;
      const paramsReplaced = this.replaceTemplate(params);

      return `${command} ${paramsReplaced}`;
    },
    replaceTemplate(params) {
      if (!params) {
        return '';
      }

      const dataMap = this.dataMap ? this.dataMap : {};
      const mapObj = {
        "{KEY}": this.redisKey,
        // "{VALUE}": this.content,
        "{FIELD}": dataMap.key,
        "{SCORE}": dataMap.score,
        "{MEMBER}": dataMap.member,
      };

      const re = new RegExp(Object.keys(mapObj).join("|"), "gi");
      const replaced = params.replace(re, matched => {
        return mapObj[matched];
      });

      return replaced;
    },
    execCommand() {
      if (!this.content || !this.content.length) {
        return this.execResult = '';
      }

      const command = this.getCommand();

      if (!command) {
        return this.execResult = 'Command Error, Check Config!';
      }

      // in case of long content
      this.previewCommand = command.replace(
        "{VALUE}",
        this.$util.cutString(this.content.toString(), this.previewContentMax)
      );
      this.fullCommand = command.replace("{VALUE}", this.content);

      try {
        shell.exec(this.fullCommand, (e, stdout, stderr) => {
          if (e || stderr) {
            this.execResult = `${e.message.trim()}\n${stdout.trim()}\n${stderr.trim()}`;
          }
          else {
            this.execResult = stdout.trim();
          }
        });
      } catch (e) {
        return this.execResult = e.message;
      }
    },
  },
  mounted() {
    this.execCommand();
  },
};
</script>

<style type="text/css">
  .text-formated-container .command-preview {
    color: #9798a7;
    word-break: break-word;
  }
</style>
