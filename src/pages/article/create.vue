<template>
  <div class="container">
    <h3 class="title is-3">发布一篇新的文章</h3>
    <el-form ref="form" :model="form" :rules="rules" label-width="80px" label-position="top">
      <el-form-item label="文章分类" prop="category">
        <el-select
          v-model="form.category"
          value-key="id"
          placeholder="请选择文章分类">
          <el-option
            v-for="cate in categories"
            :key="cate.objectId"
            :label="cate.get('name')"
            :value="cate">
          </el-option>
        </el-select>
      </el-form-item>

      <el-form-item label="文章标题" prop="title">
        <el-input v-model="form.title" placeholder="请输入文章标题"></el-input>
      </el-form-item>

      <div class="el-form-item is-required" :class="{ 'is-error': validate.error }">
        <label class="el-form-item__label">文章内容</label>
        <div class="el-form-item__content">
          <div id="editor"></div>
          <div v-if="validate.error" class="el-form-item__error">正文怎能没有内容呢？</div>
        </div>
      </div>

      <div class="oprator right">
        <el-button class="submit" type="primary" @click="submit" @keyup.enter="submit">发布文章</el-button>
      </div>

    </el-form>
  </div>
</template>

<script>

import { mapState } from 'vuex';

let editor = null;

export default {

  name: 'create',

  data () {
    return {
      categories: [],
      form: {
        category: null,
        title: '',
      },
      rules: {
        title: [
          { required: true, message: "必须填写标题哦!", trigger: 'blur' },
        ],
        category: [
          { type: 'object', required: true, message: "必须填写分类哦!", trigger: 'blur' },
        ],
      },

      validate: {
        error: false
      }
    };
  },
  created(){
    this.getCategory();
  },
  mounted(){
    this.initEditor();
    this.$Progress.finish();
  },
  computed: mapState(['user']),
  methods: {
    content(){
      return editor.$txt.html();
    },
    initEditor(){
      let E = window.wangEditor;
      editor = new E('editor')

      editor.config.menus = $.map(wangEditor.config.menus, function(item, key) {
          // https://www.kancloud.cn/wangfupeng/wangeditor2/113975 请看这里
          if (item === 'location') {
            return null;
          }
          return item;
      });

      editor.create();

      editor.onchange = () => {
        this.validateContent();
      }

    },
    getCategory(){
      const cq = new this.$api.SDK.Query('Category');
      cq.find().then((categories) => {
        this.categories = categories;
        this.form.category = categories[0];
      }).catch(console.error)
    },

    validateContent(){
      if (this.content() == '<p><br></p>') {
        this.validate.error = true;
        $('.wangEditor-container').css({borderColor:'red'})
        return;
      }

      this.validate.error = false;
      $('.wangEditor-container').css({borderColor:'#ccc'})
    },

    createArticle(){
      const article = new this.$api.SDK.Object('Article');
      article.set('author', this.user);
      article.set('title', this.form.title);
      article.set('content', this.content());
      article.set('category', this.form.category);
      return article;
    },

    setACL(article){
      // 设置访问权限
      // https://leancloud.cn/docs/acl-guide.html#单用户权限设置
      let acl = new this.$api.SDK.ACL();
      acl.setPublicReadAccess(true);
      acl.setWriteAccess(this.user,true);
      article.setACL(acl);
    },

    save(article){
      article.save().then((article) => {
        console.log(article);
        const message =  `文章《${article.get('title')}》发布成功`;

        // 同步到朋友圈状态
        const status = new this.$api.SDK.Status();
        status.inboxType = 'friend';
        status.set('title', article.get('title'));
        status.set('type', 'create_article');
        status.set('article', article);
        this.$api.SDK.Status.sendStatusToFollowers(status).then((status) => {
          //发布状态成功，返回状态信息
          console.dir(status);
        }, (err) => {
          //发布失败
          console.dir(err);
        });

        // 前端显示创建成功消息
        this.$message({message, type: 'success'})
        this.$router.replace('/article?type=all');
      }).catch(console.error);
    },

    submit(){
      this.$refs.form.validate((valid) => {
        this.validateContent();
        if (valid) {
          const article = this.createArticle();
          this.setACL(article);
          this.save(article);
        } else {
          console.log('error submit!!');
           this.$message.error('错了哦，您填写的信息有错误，请按照提示修改。');
          return false;
        }
      })

    }
  },
};
</script>

<style lang="css" scoped>
.oprator{
  margin-top: 20px;
  float: right;
}
#editor{
  min-height: 300px;
}
</style>
