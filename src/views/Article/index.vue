<template>
  <div class="article-container">
    <!-- 导航栏 -->
    <van-nav-bar class="page-nav-bar" title="文章详情">
      <template #left>
        <van-icon
          name="arrow-left"
          size="18"
          class="nav-left"
          @click="$router.back()"
        />
      </template>
      <template #right>
        <van-button plain type="default" to="/search">
          <van-icon name="search" size="18" class="nav-right" />
        </van-button>
      </template>
    </van-nav-bar>
    <div class="main-wrap">
      <!-- 加载中 -->
      <div class="loading-wrap" v-if="loading">
        <van-loading color="rgb(158, 156, 156)" vertical>加载中</van-loading>
      </div>
      <!-- /加载中 -->

      <!-- 加载完成-文章详情 -->
      <div class="article-detail" v-else-if="article.title">
        <!-- 文章标题 -->
        <h1 class="article-title">{{ article.title }}</h1>
        <!-- /文章标题 -->

        <!-- 用户信息 -->
        <van-cell class="user-info" center :border="false">
          <van-image
            class="avatar"
            slot="icon"
            round
            fit="cover"
            :src="article.aut_photo"
          >
          </van-image>
          <div slot="title" class="user-name">{{ article.aut_name }}</div>
          <div slot="label" class="publish-date">
            {{ article.pubdate | relativeTime }}
          </div>
          <!-- :is-followed="article.is_followed"
            :user-id="article.aut_id"-->
          <FollowUser
            v-model="article.is_followed"
            :user-id="article.aut_id"
            class="follow-btn"
          >
          </FollowUser>
          <!--   <van-button
            v-if="article.is_followed"
            class="follow-btn"
            round
            size="small"
            @click="onFollow"
            :loading="followLoading"
            >已关注
          </van-button>
          <van-button
            v-else
            class="follow-btn"
            type="info"
            color="#ff3a3a"
            round
            size="small"
            icon="plus"
            :loading="followLoading"
            @click="onFollow"
          >
            关注
          </van-button> -->
        </van-cell>
        <!-- /用户信息 -->

        <!-- 文章内容 -->
        <div
          class="article-content markdown-body"
          v-html="article.content"
          ref="article-content"
        ></div>
        <van-divider>正文结束</van-divider>
        <!--文章评论列表-->
        <!--   <ArticleCommentList :source="article.art_id"></ArticleCommentList> -->
        <ArticleComment
          :source="article.art_id"
          @onload-success="totalCommentCount = $event.total_count"
          :list="commentList"
          @reply-click="onReplyClick"
        >
        </ArticleComment>
        <!-- 底部区域 -->
        <div class="article-bottom">
          <van-button
            class="comment-btn"
            type="default"
            round
            size="small"
            @click="isPostShow = true"
          >
            写评论
          </van-button>
          <van-icon name="comment-o" :info="totalCommentCount" color="#777" />
          <CollectArticle
            v-model="article.is_collected"
            :article-id="article.art_id"
            class="btn-item"
          >
          </CollectArticle>
          <!--  <van-icon color="#777" name="star-o" /> -->
          <!--  <van-icon color="#777" name="good-job-o" /> -->
          <LikeArticle
            class="btn-item2"
            v-model="article.attitude"
            :article-id="article.art_id"
          >
          </LikeArticle>
          <van-icon name="share" color="#777777"></van-icon>
        </div>
        <!-- /底部区域 -->
        <!--发布评论-->
        <van-popup v-model="isPostShow" position="bottom">
          <CommentPost :target="article.art_id" @post-success="onPostSuccess">
          </CommentPost>
        </van-popup>
      </div>
      <!-- /加载完成-文章详情 -->

      <!-- 加载失败：404 -->
      <div v-else-if="errStatus === 404" class="error-wrap">
        <van-icon name="failure" />
        <p class="text">该资源不存在或已删除！</p>
      </div>
      <!-- /加载失败：404 -->

      <!-- 加载失败：其它未知错误（例如网络原因或服务端异常） -->
      <div v-else class="error-wrap">
        <van-icon name="failure" />
        <p class="text">内容加载失败！</p>
        <van-button class="retry-btn" @click="loadArticle">点击重试</van-button>
      </div>
      <!-- /加载失败：其它未知错误（例如网络原因或服务端异常） -->
    </div>
    <!--评论回复-->
    <!--弹出层是懒渲染，之后它的关闭和显示都是在切换内容的显示与隐藏-->
    <van-popup
      v-model="isReplyShow"
      position="bottom"
      :style="{ height: '95%' }"
    >
      <CommentReply
        v-if="isReplyShow"
        :comment="currentComment"
        @close="isReplyShow = false"
      >
      </CommentReply>
    </van-popup>
  </div>
</template>

<script>
import { getArticleById } from "@/api/article";
import { ImagePreview } from "vant";
import FollowUser from "@/components/follow-user/index.vue";
import CollectArticle from "@/components/collect-article/index.vue";
import LikeArticle from "@/components/like-article/index.vue";
import ArticleComment from "@/views/Article/components/article-comment";
import CommentPost from "./components/comment-post";
import CommentReply from "./components/comment-reply";

/* ImagePreview({
  images: [
    "https://img01.yzcdn.cn/vant/apple-1.jpg",
    "https://img01.yzcdn.cn/vant/apple-2.jpg",
  ],
  //起始位置
  startPosition: 1,
  onClose() {
    console.log("111");
  },
}); */
export default {
  name: "ArticleIndex",
  props: {
    articleId: {
      type: [Number, String],
      required: true,
    },
  },
  //给所有的后代组件提供数据
  provide: function () {
    return {
      articleId: this.articleId,
    };
  },
  data() {
    return {
      article: {},
      loading: true,
      errStatus: 0,
      followLoading: false,
      totalCommentCount: 0,
      isPostShow: false,
      commentList: [], //评论列表
      isReplyShow: false,
      currentComment: {}, //当前点击回复的评论项
    };
  },
  created() {
    this.loadArticle();
  },
  methods: {
    async loadArticle() {
      this.loading = true;
      try {
        const { data } = await getArticleById(this.articleId);
        //console.log(data.data.content);
        this.article = data.data;
        //请求成功，关闭loading
        //this.loading = false;
        //初始化图片点击预览
        //console.log(this.$refs["article-content"]);
        setTimeout(() => {
          //console.log(this.$refs["article-content"]);
          this.previewImage();
        }, 0);
      } catch (error) {
        //this.loading = false;
        if (error.response && error.response.status === 404) {
          this.errStatus = 404;
        }
        console.log(error);
      }
      this.loading = false;
    },
    /*   async onFollow() {
      this.followLoading = true;
      try {
        if (this.article.is_followed) {
          //已关注
          await deleteFollow(this.article.aut_id);
          //console.log(data);
          //this.article.is_followed = false;
        } else {
          //未关注
          await addFollow(this.article.aut_id);
          //console.log(data);
          // this.article.is_followed = true;
        }
        this.article.is_followed = !this.article.is_followed;
      } catch (error) {
        let message = "操作失败，请重试！";
        if (error.response && error.response.status === 400) {
          message = "你不能关注你自己!";
        }
        this.$toast(message);
      }
      this.followLoading = false;
    }, */
    async previewImage() {
      //得到所以的Image结点
      const articleContent = this.$refs["article-content"];
      const imgs = articleContent.querySelectorAll("img");
      //console.log(imgs);
      const images = [];
      imgs.forEach((img, index) => {
        images.push(img.src);
        img.onclick = () => {
          ImagePreview({
            //预览的图片地址数组
            images,
            //起始位置
            startPosition: index,
          });
        };
      });
      //console.log(images);
    },
    async onPostSuccess(data) {
      this.isPostShow = false;
      this.commentList.unshift(data.new_obj);
    },
    async onReplyClick(comment) {
      //console.log(comment);
      this.currentComment = comment;
      this.isReplyShow = true;
    },
  },
  components: {
    FollowUser,
    CollectArticle,
    LikeArticle,
    ArticleComment,
    CommentPost,
    CommentReply,
  },
};
</script>

<style lang="less" scoped>
@import "./github-markdown.css";
.article-container {
  .van-button--default {
    border: 0 solid rgb(158, 156, 156);
  }
  .page-nav-bar {
    .nav-left {
      color: black;
    }
    .nav-right {
      color: black;
    }
  }
  .main-wrap {
    position: fixed;
    left: 0;
    right: 0;
    top: 92px;
    bottom: 88px;
    overflow-y: scroll;
    background-color: #fff;
  }
  .article-detail {
    .article-title {
      font-size: 40px;
      padding: 50px 32px;
      margin: 0;
      color: #3a3a3a;
    }

    .user-info {
      padding: 0 32px;
      .avatar {
        width: 70px;
        height: 70px;
        margin-right: 17px;
      }
      .van-cell__label {
        margin-top: 0;
      }
      .user-name {
        font-size: 24px;
        color: #3a3a3a;
      }
      .publish-date {
        font-size: 23px;
        color: #b7b7b7;
      }
      .follow-btn {
        width: 170px;
        height: 58px;
      }
    }

    .article-content {
      padding: 55px 32px;
      /deep/ p {
        text-align: justify;
      }
    }
    /deep/ .btn-item {
      width: 20px;
      height: 20px;
      color: rgb(119, 119, 119);
      font-size: 33px;
    }
    /deep/ .btn-item2 {
      width: 20px;
      height: 20px;
      color: rgb(119, 119, 119);
      font-size: 34px;
    }
  }

  .loading-wrap {
    padding: 200px 32px;
    display: flex;
    align-items: center;
    justify-content: center;
    background-color: #fff;
  }

  .error-wrap {
    padding: 200px 32px;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    background-color: #fff;
    .van-icon {
      font-size: 122px;
      color: #b4b4b4;
    }
    .text {
      font-size: 30px;
      color: #666666;
      margin: 33px 0 46px;
    }
    .retry-btn {
      width: 280px;
      height: 70px;
      line-height: 70px;
      border: 1px solid #c3c3c3;
      font-size: 30px;
      color: #666666;
    }
  }

  .article-bottom {
    position: fixed;
    left: 0;
    right: 0;
    bottom: 0;
    display: flex;
    justify-content: space-around;
    align-items: center;
    box-sizing: border-box;
    height: 88px;
    border-top: 1px solid #d8d8d8;
    background-color: #fff;
    .comment-btn {
      width: 282px;
      height: 46px;
      border: 2px solid #eeeeee;
      font-size: 30px;
      line-height: 46px;
      color: #a7a7a7;
    }
    .van-icon {
      font-size: 40px;
      .van-info {
        font-size: 16px;
        background-color: #e22829;
      }
    }
  }
}
</style>
