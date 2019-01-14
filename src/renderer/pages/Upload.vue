<template>
  <div class="upload-page">
    <bucket-header></bucket-header>
    <!-- manage tool -->
    <div class="manage-tool">
      <div class="bucket-info">
        <el-tag class="bucket-name">{{ bucket }}</el-tag>
      </div>
      <div class="manage-btn">
        <el-button class="w-btn" type="text" icon="arrow-left" @click="goback()">返回</el-button>
        <div class="one-handle">
          <span class="w-btn label">粘贴上传：</span>
          <el-switch
            v-model="canPasteUpload"
            on-text="启用"
            off-text="禁用">
          </el-switch>
        </div>
          <div class="one-handle">
              <span class="w-btn label">路径前缀：</span>
              <el-input
                      placeholder="可以用来分类文件，默认为空"
                      v-model="prefixName"
                      clearable>
              </el-input>
          </div>
          <div class="one-handle">
              <span class="w-btn label">命名方式：</span>
              <el-select v-model="renameMode">
                  <el-option
                          label="原文件名"
                          :value="0">
                  </el-option>
                  <el-option
                          label="手动输入"
                          :value="1">
                  </el-option>
                  <el-option
                          label="自动生成（时间戳）"
                          :value="2">
                  </el-option>
              </el-select>
          </div>
      </div>
    </div>

    <div class="upload-panel">
      <el-upload
        class="upload-demo"
        :action="uploadUrl"
        :file-list="uploadFileList"
        drag
        :on-remove="handleRemove"
        :before-upload="beforeUpload"
        :on-success="handleSuccess"
        :on-error="handleError"
        :on-progress="handleProgress"
        :data="form">
        <i class="el-icon-upload"></i>
        <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
      </el-upload>
    </div>
  </div>
</template>

<script>
  import BucketHeader from '../components/BucketHeader';
  import PutPolicy from '../utils/put_policy';
  import Qiniu from '../utils/qiniu';

  export default {
    name: 'upload',
    components: { BucketHeader },
    data() {
      return {
        canPasteUpload: true,
        renameMode: 0,
        prefixName: '',
        uploadUrl: '',
        bucket: this.$route.query.bucket,
        form: {},
        headers: {},
        uploadFileList: [],
      };
    },
    created() {
      Qiniu.autoZone(localStorage.accessKey, this.bucket)
        .then((zone) => {
          this.uploadUrl = `http://${zone.up.src.main[0]}`;
        })
        .catch();
      document.addEventListener('paste', this.pasteEvent);
    },
    beforeDestroy() {
      document.removeEventListener('paste', this.pasteEvent);
    },
    methods: {
      pasteEvent(event) {
        const items = event.clipboardData && event.clipboardData.items;
        let file = null;
        console.log(this.uploadFileList);
        if (!this.canPasteUpload) {
          return;
        }
        if (items && items.length) {
          // 检索剪切板items
          for (let i = 0; i < items.length; i += 1) {
            if (items[i].type.indexOf('image') !== -1) {
              file = items[i].getAsFile();
              break;
            }
          }
        }
        // 检查是否需要上传文件
        if (file) {
          this.$message('从剪切板上传文件...📋');
          this.pasteUpload(file);
        }
      },
        pasteUpload(file) {
            return this.getUploadInfo(file).then(({name, token}) => {
                const formData = new FormData();

                // 文件
                formData.append('file', file);

                // 其他些参数
                formData.append('key', name);
                formData.append('token', token);

                // ajax上传
                const xhr = new XMLHttpRequest();
                // 上传结束
                xhr.onload = () => {
                    let res = null;
                    try {
                        res = JSON.parse(xhr.responseText);
                        this.handleSuccess(res);
                    } catch (e) {
                        console.error(e);
                    }
                };
                xhr.open('POST', this.uploadUrl, true);
                xhr.send(formData);
            });
        },
      // 生成上传token
      generateUploadToken(fileName) {
        const options = {
          scope: `${this.bucket}:${fileName}`,
        };
        const mac = {
          accessKey: localStorage.accessKey,
          secretKey: localStorage.secretKey,
        };
        const putPolicy = new PutPolicy(options);
        return putPolicy.uploadToken(mac);
      },
      goback() {
        this.$router.push({ path: `/manage?bucket=${this.bucket}` });
      },
      handleSuccess(res) {
        this.$message('上传成功');
        this.uploadFileList.push({
            ...res,
            'name': res.key,
        });
      },
      handleError(err) {
          this.$message.error(`上传失败：${err}`);
      },
      handleProgress() {
      },
      handleRemove(item) {
        const bucket = this.$route.query.bucket;
        const accessKey = localStorage.accessKey;
        const secretKey = localStorage.secretKey;
        Qiniu.delete(accessKey, secretKey, bucket, item.key)
          .then(() => {
            this.$message('删除成功...💗');
            this.uploadFileList = this.uploadFileList.filter((one)=>{
                return one.key !== item.key;
            })
          })
          .catch();
      },
        getUploadInfo(file) {
          let p = null;
          switch (this.renameMode) {
              // 手动输入文件名
              case 1: {
                  p = this.$prompt('请输入新的文件名', '提示', {
                      inputValue: 'file.name',
                      confirmButtonText: '确定并上传',
                      cancelButtonText: '取消',
                      inputPattern: /^[\s\S]*.*[^\s][\s\S]*$/,
                      inputErrorMessage: '请输入正确的文件名'
                  }).then(({value}) => value);
                  break;
              }
              // 使用时间戳自动重命名
              case 2: {
                  p = Promise.resolve(`arn_${Date.now()}_${file.name}`);
                  break;
              }
              default: {
                  p = Promise.resolve(file.name);
              }
          }

          return p.then((name) => {
              // 设置分组名字
              const fullName = `${this.prefixName}${name}`;
              return {
                  token: this.generateUploadToken(fullName),
                  name: fullName
              }
          });
        },
      async beforeUpload(file) {
          // form data
          return this.getUploadInfo(file).then(({name, token}) => {
              this.form.token = token;
              this.form.key = name;
          });
      },
    },
  };
</script>

<style>
  webkit,
  ::-webkit-scrollbar {
    width: 0;
  }
  body {
    margin: 0;
    font-family: "Helvetica Neue",Helvetica,"PingFang SC","Hiragino Sans GB","Microsoft YaHei","微软雅黑",Arial,sans-serif;
  }
  .upload-panel {
    padding-top: 100px;
  }
  .manage-tool {
    position: fixed;
    margin-top: 50px;
    width: 100vw;
    height: 50px;
    -webkit-app-region: drag;
    background: #2e84c7;
  }
  .bucket-info {
    float: left;
    margin-left: 10vw;
    padding-top: 4px;
  }
  .bucket-name {
    background: #fff;
    color: #2e84c7;
  }
  .manage-btn {
    float: left;
    margin-left: 4vw;
  }
  .w-btn,
  .w-btn:hover,
  .w-btn:focus {
    color: #fff;
  }
  /* dtrag upload style */
  .el-upload {
    float: right;
  }
  .el-upload-dragger {
    width: 61vw;
    height: 536px;
    border: 0;
    border-left: 1px solid #eee;
    border-radius: 0;
    background: transparent;
  }
  .el-upload-dragger:hover {
    border: 1px dashed #2e84c7;
  }
  .el-upload-dragger .el-icon-upload {
    margin-top: 30vh;
  }
  .el-upload-list {
    position: absolute;
    width: 35vw;
    top: 100px;
    left: 2vw;
    height: 540px;
    overflow: scroll;
  }
  .one-handle {
    display: inline-block;
    margin-left: 20px;
    font-size: 14px;
  }
  .one-handle .el-input {
    width: 208px;
  }
  .one-handle .el-select .el-input{
      width: 168px;
  }

</style>
