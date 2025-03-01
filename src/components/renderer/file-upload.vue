<template>
  <div>
    <label v-uni-for="name">{{ label }}</label>
    <b-card v-if="inPreviewMode" class="mb-2">
      {{ $t('File uploads are unavailable in preview mode.') }}
    </b-card>
    <uploader
      v-else
      :options="options"
      :attrs="attrs"
      ref="uploader"
      @complete="complete"
      @upload-start="start"
      @file-removed="removed"
      @file-success="fileUploaded"
      @file-added="addFile"
      :class="{'was-validated': required}"
    >
      <uploader-unsupport/>

      <uploader-drop class="form-control-file">
        <p>{{ $t('Drop a file here to upload or') }}</p>
        <uploader-btn tabindex="0" v-on:keyup.native="browse" :aria-label="$attrs['aria-label']" class="btn btn-secondary text-white">{{ $t('select file') }}</uploader-btn>
        <span v-if="validation === 'required' && !value" class="required">{{ $t('Required') }}</span>
      </uploader-drop>
      <uploader-list>
        <template slot-scope="{ fileList }">
          <ul>
            <li v-if="fileList.length === 0 && value">
              <div class="border-bottom py-2">
                <i class="fas fa-paperclip"/> {{ displayName }}
              </div>
            </li>
            <li v-for="file in fileList" :key="file.id">
              <uploader-file :file="file" :list="true"/>
            </li>
          </ul>
        </template>
      </uploader-list>
      <div class="invalid-feedback" :class="{'d-block': required && !value}">
        {{ $t('Field is required') }}
      </div>
    </uploader>

    <div class="invalid-feedback" v-if="error">{{ error }}</div>
    <small v-if="helper" class="form-text text-muted">{{ helper }}</small>
  </div>
</template>

<script>
import { createUniqIdsMixin } from 'vue-uniq-ids';
import uploader from 'vue-simple-uploader';
import _ from 'lodash';

// Create the mixin
const uniqIdsMixin = createUniqIdsMixin();

// vue-simple-uploader tries to call these after the component has
// been destroyed since it does it in nextTick(). It has no effect
// on functionality because a new copy is created. TODO: Why is this
// component being recreated when used in a loop?
const ignoreErrors = [
  'Cannot read property \'assignBrowse\' of null',
  'Cannot read property \'assignDrop\' of null',
];

export default {
  components: uploader,
  mixins: [uniqIdsMixin],
  props: ['label', 'error', 'helper', 'name', 'value', 'controlClass', 'endpoint', 'accept', 'validation', 'parent', 'config'],
  beforeMount() {
    this.getFileType();
  },
  updated() {
    this.removeDefaultClasses();
  },
  mounted() {
    this.$root.$on('set-upload-data-name',
      (recordList, index, id) => this.listenRecordList(recordList, index, id));

    this.$root.$on('removed-record',
      (recordList, record) => this.listenRemovedRecord(recordList, record));

    this.$root.$on('removed-loop',
      (loop, removed) => this.listenRemovedLoop(loop, removed));

    this.removeDefaultClasses();
    
    this.checkIfInRecordList();

    this.setPrefix();
    if (this.$refs['uploader']) {
      this.$refs['uploader'].$forceUpdate();
    }
  },
  errorCaptured(err) {
    if (ignoreErrors.includes(err.message)) {
      return false;
    }
  },
  computed: {
    required() {
      if (this.config && this.config.validation) {
        return this.config.validation === 'required';
      } else {
        return false;
      }
    },
    inPreviewMode() {
      return ((this.mode === 'preview' && !window.exampleScreens) || this.mode === 'editor');
    },
    displayName() {
      const requestFiles = _.get(window, 'PM4ConfigOverrides.requestFiles', {});
      const fileInfo = requestFiles[this.fileDataName];
      let id = this.uploaderId;
      if (fileInfo && id >= 0) {
        return fileInfo.file_name;
      }
      return this.value.name ? this.value.name : this.value;
    },
    mode() {
      return this.$root.$children[0].mode;
    },
    classList() {
      return {
        'is-invalid': (this.validator && this.validator.errorCount) || this.error,
        [this.controlClass]: !!this.controlClass,
      };
    },
    inProgress() {
      return this.$refs.uploader.fileList.some(file => file._prevProgress < 1);
    },
    filesAccept() {
      if (!this.accept) {
        return null;
      }

      let accept = [];

      (this.accept.split(',')).forEach(item => {
        accept.push(item.trim());
      });
      return accept;
    },
    fileDataName() {
      return this.prefix + this.name + (this.row_id ? '.' + this.row_id : '');
    },
  },
  watch: {
    name: {
      handler() {
        this.options.query.data_name = this.fileDataName;
      },
      immediate: true,
    },
    parent: {
      handler() {
        this.options.query.parent = this.parent;
      },
      immediate: true,
    },
    prefix: {
      handler() {
        this.options.query.data_name = this.fileDataName;
      },
      immediate: true,
    },
    row_id: {
      handler() {
        this.options.query.row_id = this.row_id;
        this.options.query.data_name = this.prefix + this.name + (this.row_id ? '.' + this.row_id : '');
      },
      immediate: true,
    },
  },
  data() {
    return {
      uploaderId: 1,
      content: '',
      fileType: null,
      validator: {
        errorCount: 0,
        errors: [],
      },
      prefix: '',
      row_id: null,
      options: {
        target: this.getTargetUrl,
        // We cannot increase this until laravel chunk uploader handles this gracefully
        simultaneousUploads: 1,
        query: {
          chunk: true,
          data_name: this.name,
          parent: null,
          row_id: null,
        },
        testChunks: false,
        // Setup our headers to deal with API calls
        headers: {
          'X-Requested-With': 'XMLHttpRequest',
          'X-CSRF-TOKEN': window.ProcessMaker.apiClient.defaults.headers.common['X-CSRF-TOKEN'],
        },
        singleFile: true,
      },
      attrs: {
        accept: this.accept,
      },
    };
  },
  methods: {
    listenRemovedLoop(loop, removed) {
      this.deleteAssociatedFiles(removed);
    },
    listenRemovedRecord(recordList, record) {
      const parent = this.parentRecordList(this);
      if (parent !== recordList) {
        return;
      }
      this.deleteAssociatedFiles(record);
    },
    deleteAssociatedFiles(object) {
      for (const prop in object) {
        if (prop === this.name) {
          this.deleteFile(object[prop]);
        }
        if (Array.isArray(object[prop])) {
          for (const item of object[prop]) {
            this.deleteAssociatedFiles(item);
          }
        }
      }
    },
    deleteFile(fileId) {
      if (fileId) {
        window.ProcessMaker.apiClient
          .delete(`files/${fileId}`)
          .catch(() => {
            /** ignore exception **/
          });
      }
    },
    listenRecordList(recordList, index, id) {
      const parent = this.parentRecordList(this);
      if (parent !== recordList) {
        return;
      }
      this.row_id = (parent !== null) ? id : null;
      //update id to refresh computed values
      this.uploaderId =new Date().getTime();
      if (this.$refs.uploader) {
        this.$refs.uploader.files = [];
        this.$refs.uploader.fileList = [];
        this.$refs.uploader.uploader.files = [];
        this.$refs.uploader.uploader.fileList = [];
      }
      this.$forceUpdate();
    },
    setPrefix() {
      let parent = this.$parent;
      let i = 0;
      while (!parent.loopContext) {
        parent = parent.$parent;

        if (parent === this.$root) {
          parent = null;
          break;
        }

        i++;
        if (i > 100) {
          throw 'Loop Error';
        }
      }

      if (parent && parent.loopContext) {
        this.prefix = parent.loopContext + '.';
      }
    },
    setFileUploadNameForChildren(children, prefix) {
      children.forEach(child => {
        if (_.get(child, '$options.name') === 'FileUpload') {
          child.prefix = prefix;
        } else if (_.get(child, '$children', []).length > 0) {
          this.setFileUploadNameForChildren(child.$children, prefix);
        }
      });
    },
    addFile(file) {
      if (this.filesAccept) {
        file.ignored = true;
        if (this.filesAccept.indexOf(file.fileType) !== -1) {
          file.ignored = false;
        }
        if (file.ignored) {
          window.ProcessMaker.alert(this.$t('File not allowed.'), 'danger');
          return false;
        }
      }
      file.ignored = false;
      if (!this.name) {
        this.options.query.data_name = file.name;
      }
      return true;
    },
    removeDefaultClasses() {
      // we need to be able to remove the classes from the npm package
      this.$el.querySelectorAll('.uploader-btn, .uploader-drop').forEach(el => {
        el.classList.remove('uploader-btn', 'uploader-drop');
      });
    },
    browse(e) {
      if (['Enter', 'Space'].includes(e.code)) {
        e.target.click();
      }
    },
    getFileType() {
      if (document.head.querySelector('meta[name="collection-id"]')) {
        this.fileType = 'collection';
      } else {
        this.fileType = 'request';
      }
    },
    fileUploaded(rootFile, file, message) {
      if (this.fileType == 'request') {
        let id = '';
        if (message) {
          const msg = JSON.parse(message);
          if (!_.has(window, 'PM4ConfigOverrides')) {
            window.PM4ConfigOverrides = {};
          }
          if (!_.has(window, 'PM4ConfigOverrides.requestFiles')) {
            window.PM4ConfigOverrides.requestFiles = {};
          }
          window.PM4ConfigOverrides.requestFiles[this.fileDataName] = { id:msg.fileUploadId, file_name:file.name };
          id = msg.fileUploadId;
        }
        this.$emit('input', id);
      }

      if (this.fileType == 'collection') {
        message = JSON.parse(message);
        this.$emit('input', {
          id: message.id,
          name: message.file_name,
        });
      }
    },
    removed() {
      if (!this.inProgress) {
        this.complete();
      }
    },
    complete() {
      // Unblock submit
      this.validator.errorCount = 0;
      window.onbeforeunload = function() {};
    },
    parentRecordList(node) {
      if (node.$parent && node.$parent.$options) {
        if (node.$parent.$options._componentTag ===  'form-record-list') {
          return node.$parent;
        }
        return this.parentRecordList(node.$parent);
      }
      return null;
    },
    start() {
      if (this.parentRecordList(this) === null) {
        this.row_id = null;
      }

      // Block submit until files are loaded
      this.validator.errorCount = 1;
      window.onbeforeunload = function() {
        return true;
      };
    },
    getTargetUrl() {
      if (_.has(window, 'PM4ConfigOverrides.postFileEndpoint')) {
        return window.PM4ConfigOverrides.postFileEndpoint;
      }
      
      if (this.endpoint) {
        return this.endpoint;
      }

      if (this.fileType == 'request') {
        const requestIDNode = document.head.querySelector('meta[name="request-id"]');

        return requestIDNode
          ? `/api/1.0/requests/${requestIDNode.content}/files`
          : null;
      }

      if (this.fileType == 'collection') {
        const collectionIdNode = document.head.querySelector('meta[name="collection-id"]');

        return collectionIdNode
          ? '/api/1.0/files' +
            '?model=' +
            'ProcessMaker\\Plugins\\Collections\\Models\\Collection' +
            '&model_id=' +
            collectionIdNode.content +
            '&collection=' +
            'collection'
          : null;
      }
    },
    checkIfInRecordList() {
      const parent = this.parentRecordList(this);
      if (parent !== null) {
        const recordList = parent;
        const prefix = recordList.name + '.';
        this.setFileUploadNameForChildren(recordList.$children, prefix);
      }
    },
  },
};
</script>

<style scoped>
.required {
  color: red;
  font-size: 0.8em;
}
</style>
