<template>
  <div class="container">
    <el-row>
      <h2 style="text-align: center;
    margin: 30px;">{{currentPage.name}}</h2>
    </el-row>
    <el-form :model="formModal" ref="ruleForm" label-width="100px" class="demo-ruleForm">
      <el-form-item label="文章标题：">
        <el-input v-model="formModal.title"></el-input>
      </el-form-item>
      <el-form-item label="文章分类：">
        <el-select v-model="formModal.type" placeholder="选择分类">
          <el-option
            v-for="(item,index) in typeOpitons"
            :key="'cate'+index"
            :label="item.name"
            :value="item.id"
          />
        </el-select>
      </el-form-item>
      <el-form-item label="关键词：">
        <el-input v-model="formModal.keywords" @keydown.enter.native="keywords_actions('add',null)"></el-input>
        <div>
          <span
            class="keywords_g"
            @click="keywords_actions('delete',item)"
            v-for="(item,indexk) in keywords_g"
            :key="'keywords'+indexk"
          >{{item}}</span>
        </div>
      </el-form-item>
      <el-form-item label="文章来源：">
        <el-input v-model="formModal.source"></el-input>
      </el-form-item>
      <el-form-item label="文章头图：">
        <el-upload
          class="upload-demo"
          :before-upload="beforeupload"
          action
          :multiple="false"
          :show-file-list="false"
          accept=".png, .jpg, .jpeg"
        >
          <el-button size="small" type="primary">点击上传</el-button>
        </el-upload>
        <div class="upload_img">
          <img v-if="upload_img" :src="upload_img" alt />
        </div>
      </el-form-item>
      <el-form-item label="头图描述：">
        <el-input v-model="formModal.headImgDesc"></el-input>
      </el-form-item>
      <el-form-item label="摘要：">
        <el-input type="textarea" :rows="5" placeholder="请输入内容" v-model="formModal.summary"></el-input>
      </el-form-item>
      <el-form-item label="正文：">
        <quill-editor v-model="formModal.content" ref="myQuillEditor" :options="editorOption"></quill-editor>
      </el-form-item>

      <!-- // -->
      <el-form-item>
        <el-button type="primary" @click="submit">{{currentPage.button}}</el-button>
        <el-button @click="route_back">返回</el-button>
      </el-form-item>
    </el-form>
    <el-upload
      v-show="false"
      id="quill-upload"
      :action="''"
      name="upload_file"
      multiple
      :limit="100"
      accept=".png, .jpg, .jpeg"
      list-type="picture"
      :show-file-list="false"
      :before-upload="beforeuploadlist"
    >
      <el-button size="small" type="primary"></el-button>
      <div slot="tip" class="el-upload__tip">只能上传jpg/png文件，且不超过500kb</div>
    </el-upload>
  </div>
</template>


<script>
import {
  fileUpload,
  fetchCategoryListForChose,
  createArticle,
  getArticleDetail,
  editArticle
} from "@/api/article-manage";
import lrz from "lrz";
export default {
  created() {
    let query = this.$route.query;
    if (!query.code) return;
    this.currentPage = this.page.filter(x => x.code === query.code)[0];
    if (query.code === "edit") {
      if (!query.id) return;
      this.artId = query.id;
      this.getArt();
    }
    this.getCateList();
  },
  methods: {
    async getCateList() {
      let res = await fetchCategoryListForChose();
      if (res.code == "000000") {
        this.typeOpitons = res.data;
      }
    },
    async getArt() {
      let res1 = await getArticleDetail(this.artId);
      let res = res1.data;
      this.formModal.title = res.title;
      this.formModal.type = res.type;
      this.formModal.source = res.source;
      this.formModal.headImgDesc = res.headImgDesc;
      this.formModal.summary = res.summary;
      this.formModal.content = res.content;
      this.keywords_g = res.keywords ? res.keywords : [];
      this.upload_img = res.headImgUrl;
    },
    async submit() {
      let req = {
        title: this.formModal.title,
        type: this.formModal.type,
        keywords: this.keywords_g,
        source: this.formModal.source,
        headImgUrl: this.upload_img,
        headImgDesc: this.formModal.headImgDesc,
        summary: this.formModal.summary,
        content: this.formModal.content
      };
      let data = new FormData();
      for (let key in req) {
        if (!req[key]) return this.$message.info("参数不能为空");
        data.append(key, req[key]);
      }
      if (this.currentPage.code === "add") {
        let res = await createArticle(data);
        if (res.code == "000000") {
          this.$message.success(res.msg);
          this.$store
            .dispatch("tagsView/delView", this.$route)
            .then(({ visitedViews }) => {
              this.$router.push("list");
              // this.toLastView(visitedViews, this.$route);
            });
        }
      }
      if (this.currentPage.code === "edit") {
        let res = await editArticle(this.artId, data);
        if (res.code == "000000") {
          this.$message.success(res.msg);
          this.$store
            .dispatch("tagsView/delView", this.$route)
            .then(({ visitedViews }) => {
              this.$router.push("list");
              // this.toLastView(visitedViews, this.$route);
            });
        }
      }
    },

    toLastView(visitedViews, view) {
      const latestView = visitedViews.slice(-1)[0];
      if (latestView) {
        this.$router.push(latestView.fullPath);
      } else {
        // now the default is to redirect to the home page if there is no tags-view,
        // you can adjust it according to your needs.
        if (view.name === "Dashboard") {
          // to reload home page
          this.$router.replace({ path: "/redirect" + view.fullPath });
        } else {
          this.$router.push("/");
        }
      }
    },
    route_back() {
      this.$router.go(-1);
    },

    async beforeupload(f) {
      // this.imageToBase64(f);
      // this.imgUpload(f);
      let res = await this.imagezip(f);
      let imgurl = await this.imgUpload(res.file);
      return false;
    },
    async beforeuploadlist(f) {
      console.log(f); //上传图片之前开启loading
      let res = await this.imagezip(f);
      console.log(res);
      let imgurl = await this.imgUpload(res.data.file,res.name);
      console.log(imgurl);
      let quill = this.$refs.myQuillEditor.quill;
      let length = quill.getSelection().index;
      // 插入图片  response.data.url为服务器返回的图片地址
      quill.insertEmbed(length, "image", imgurl);
      // 调整光标到最后
      quill.setSelection(length + 1);
      console.log(this.formModal.content)
      return false;
    },
    uploadError() {
      //图片上传失败,关闭loading
      this.$message.error("图片插入失败");
    },
    async beforeupload(f) {
      // this.imageToBase64(f);
      // this.imgUpload(f);
      let res = await this.imagezip(f);
      let imgurl = await this.imgUpload(res.data.file,res.name);
      this.upload_img = imgurl;
      return false;
    },
    imagezip(f) {
      let name = f.name;
      return new Promise((resolve, reject) => {
        lrz(f)
          .then(rst => {
            // 处理成功会执行
            // rst.file.name = name;
            console.log(rst);
            resolve({
              data:rst,
              name:name
            });
          })
          .catch(err => {
            console.log(err);
            this.$message(err)
            // reject(err);
            // 处理失败会执行
          });
      });
    },
    async imgUpload(f,name) {
      let data = new FormData();
      data.append("file", f);
      data.append("filePath", "images");
      data.append("suffix", name.split(".")[1]);
      let res = await fileUpload(data);
      if (res.code == "000000") {
        return res.data;
      }
    },
    keywords_actions(item, value) {
      if (item === "add") {
        if (!this.formModal.keywords) return;
        this.keywords_g.push(this.formModal.keywords);
        this.formModal.keywords = "";
      }
      if (item === "delete") {
        let index = this.keywords_g.indexOf(value);
        this.keywords_g.splice(index, 1);
      }
    }
  },
  data() {
    return {
      artId: "",
      editorOption: {
        modules: {
          toolbar: {
            container: [
              ["bold", "italic", "underline", "strike"], //加粗，斜体，下划线，删除线
              [{ header: 1 }, { header: 2 }], // 标题，键值对的形式；1、2表示字体大小
              [{ indent: "-1" }, { indent: "+1" }], // 缩进
              [{ size: ["small", false, "large", "huge"] }],
              [{ header: [1, 2, 3, 4, 5, 6, false] }],
              [{ color: [] }, { background: [] }], // 字体颜色，字体背景颜色
              [{ align: [] }], //对齐方式
              ["clean"], //清除字体样式
              ["link"],
              ["image"]
            ],
            handlers: {
              image: function(value) {
                if (value) {
                  console.log(value);
                  // 给个点击触发Element-ui，input框选择图片文件
                  document.querySelector("#quill-upload input").click();
                } else {
                  this.quill.format("image", false);
                }
              }
            }
          }
        }
      },
      page: [
        {
          name: "新增文章",
          code: "add",
          button: "提交"
        },
        {
          name: "编辑文章",
          code: "edit",
          button: "保存"
        }
      ],
      currentPage: {
        name: "新增文章",
        code: "add",
        button: "提交"
      },
      formModal: {
        title: "",
        type: "",
        source: "",
        keywords: "",
        headImgDesc: "",
        summary: "",
        content: ""
      },
      upload_img: "",
      keywords_g: [],
      typeOpitons: []
    };
  }
};
</script>

<style lang="scss" scoped>
.demo-ruleForm {
  width: 85%;
  margin: auto;
}
/deep/ .el-form-item__content {
  line-height: unset;
}
/deep/ .ql-editor {
  height: 500px;
}
.keywords_g {
  background-color: #269e46;
  color: #fff;
  font-size: 13px;
  padding: 0px 10px;
  border-radius: 3px;
  margin-right: 5px;
  display: inline-block;
  line-height: 26px;
  cursor: pointer;
  margin-top: 10px;
}
.upload_img {
  margin: 20px 0;
  img {
    width: 600px;
    border: 1px solid #eee;
    border-radius: 5px;
  }
}
</style>