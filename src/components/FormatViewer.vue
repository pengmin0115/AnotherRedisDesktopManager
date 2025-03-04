<template>
  <div class="format-viewer-container">
    <el-select v-model="selectedView" :disabled='overSize' class='format-selector' :style='selectStyle' size='mini' placeholder='Text'>
      <span slot="prefix" class="fa fa-sitemap"></span>
      <el-option
        v-for="item of viewers"
        :key="item.text"
        :label="item.text"
        :value="item.text">
      </el-option>
      <!-- add custom -->
      <el-option
        @click.native.stop.prevent='addCustomFormatter'
        value='addCustomFormatter'>
        <el-button type='text' icon="el-icon-edit-outline">{{$t('message.custom')}}</el-button>
      </el-option>
    </el-select>
    <span @click='copyContent' :title='$t("message.copy")' class='el-icon-document formater-copy-icon'></span>
    <span v-if='!contentVisible' class='formater-binary-tag'>[Hex]</span>
    <span class='formater-binary-tag'>Size: {{ $util.humanFileSize(buffSize) }}</span>
    <br>

    <component
      ref='viewer'
      :is='viewerComponent'
      :content='content'
      :name="selectedView"
      :contentVisible='contentVisible'
      :textrows='textrows'
      :disabled='disabled'
      :redisKey="redisKey"
      :dataMap="dataMap"
      @updateContent="updateContent">
    </component>
  </div>
</template>

<script type="text/javascript">
import storage from '@/storage';
import ViewerText from '@/components/ViewerText';
import ViewerHex from '@/components/ViewerHex';
import ViewerJson from '@/components/ViewerJson';
import ViewerBinary from '@/components/ViewerBinary';
import ViewerUnserialize from '@/components/ViewerUnserialize';
import ViewerMsgpack from '@/components/ViewerMsgpack';
import ViewerOverSize from '@/components/ViewerOverSize';
import ViewerCustom from '@/components/ViewerCustom';

export default {
  data() {
    return {
      viewerComponent: 'ViewerText',
      selectedView: 'Text',
      viewers: [
        { value: 'ViewerText', text: 'Text' },
        { value: 'ViewerHex', text: 'Hex' },
        { value: 'ViewerJson', text: 'Json' },
        { value: 'ViewerBinary', text: 'Binary' },
        { value: 'ViewerMsgpack', text: 'Msgpack' },
        { value: 'ViewerUnserialize', text: 'Unserialize' },
      ],
      selectStyle: {
        float: this.float,
      },
      overSizeBytes: 20971520, // 20MB
      manualUpdate: false,
    };
  },
  components: {ViewerText, ViewerHex, ViewerJson, ViewerBinary, ViewerUnserialize, ViewerMsgpack, ViewerOverSize, ViewerCustom},
  props: {
    float: {default: 'right'},
    content: {default: () => Buffer.from('')},
    textrows: {default: 6},
    disabled: {type: Boolean, default: false},
    redisKey:  {default: () => Buffer.from('')},
    dataMap: {type: Object, default: () => {}},
  },
  computed: {
    contentVisible() {
      // for better performance, oversize doesn't care visible.
      if (this.overSize) {
        return true;
      }
      return this.$util.bufVisible(this.content);
    },
    buffSize() {
      return Buffer.byteLength(this.content);
    },
    overSize() {
      return this.buffSize > this.overSizeBytes;
    },
    viewersMap() {
      // add oversize tmp
      let map = {OverSize: 'ViewerOverSize'};

      this.viewers.forEach(item => {
        map[item.text] = item.value;
      });

      return map;
    },
  },
  created() {
    this.$bus.$on('refreshViewers', () => {
      this.removeCustom();
      this.loadCustomViewers();
    });
  },
  watch: {
    content() {
      // do not auto format while user editting
      if (this.manualUpdate) {
        return this.manualUpdate = false;
      }

      this.autoFormat();
    },
    selectedView(viewer) {
      // custom viewer com may same, force change
      this.viewerComponent = '';
      this.$nextTick(() => {
        this.viewerComponent = this.viewersMap[viewer];
      });
    },
  },
  methods: {
    // update by user edit
    updateContent(content) {
      this.manualUpdate = true;
      this.$emit('update:content', content);
    },
    changeViewer(viewer) {
      this.selectedView = viewer;
      this.viewerComponent = this.viewersMap[viewer];
    },
    addCustomFormatter() {
      this.$bus.$emit('addCustomFormatter');
      this.autoFormat();
    },
    autoFormat() {
      if (!this.content || !this.content.length) {
        return this.changeViewer('Text');
      }

      if (this.overSize) {
        return this.changeViewer('OverSize');
      }

      // json
      if (this.$util.isJson(this.content)) {
        return this.changeViewer('Json');
      }
      // php unserialize
      else if (this.$util.isPHPSerialize(this.content)) {
        return this.changeViewer('Unserialize');
      }
      // msgpack
      else if (this.$util.isMsgpack(this.content)) {
        return this.changeViewer('Msgpack');
      }
      // hex
      else if (!this.contentVisible) {
        return this.changeViewer('Hex');
      }
      else {
        return this.changeViewer('Text');
      }
    },
    copyContent() {
      this.$util.copyToClipboard(this.content);
      this.$message.success(this.$t('message.copy_success'));
    },
    loadCustomViewers() {
      const formatters = storage.getCustomFormatter();

      if (!formatters || !formatters.length) {
        return;
      }

      formatters.forEach(formatter => {
        this.viewers.push({value: 'ViewerCustom', text: formatter.name, type: 'custom'});
      });
    },
    removeCustom() {
      this.viewers = this.viewers.filter(item => {
        return item.type !== 'custom';
      });
    },
  },
  mounted() {
    this.autoFormat();
    this.loadCustomViewers();
  },
};
</script>

<style type="text/css">
  .format-selector {
    width: 122px;
  }
  .format-selector .el-input__inner {
    height: 22px !important;
  }

  .text-formated-container {
    border: 1px solid #dcdfe6;
    min-height: 114px;
    padding: 5px 15px;
    line-height: 1.5;
    border-radius: 5px;
  }
  .text-formated-container * {
    font-family: inherit !important;
  }
  .dark-mode .text-formated-container {
    border-color: #7f8ea5;
  }

  /*json tree*/
  .dark-mode .jv-container.jv-light {
    background: none;
  }
  .dark-mode .jv-container.jv-light .jv-key {
    color: #ebebec;
  }
  .dark-mode .jv-container.jv-light .jv-item.jv-array,
  .dark-mode .jv-container.jv-light .jv-item.jv-object {
    color: #b6b6b9;
  }
  .dark-mode .jv-container.jv-light .jv-ellipsis {
    background: #c5c5c5;
  }

  .collapse-container {
    height: 27px;
  }

  .collapse-container .collapse-btn {
    float: right;
    padding: 9px 0;
  }
  .formater-binary-tag {
    /*padding-left: 5px;*/
    color: #7ab3ef;
    font-size: 80%;
  }
  .formater-copy-icon {
    color: #7ab3ef;
    cursor: pointer;
  }
</style>
