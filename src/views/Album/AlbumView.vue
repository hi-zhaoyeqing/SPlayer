<template>
  <div class="playlist" v-if="albumDetail">
    <div class="left">
      <div class="cover">
        <n-avatar
          class="coverImg"
          :src="getCoverUrl(albumDetail.picUrl)"
          fallback-src="/images/pic/default.png"
        />
        <img src="/images/pic/album.png" class="album" alt="album" />
      </div>
      <div class="meta">
        <div class="title">
          <n-text class="name">{{ albumDetail.name }}</n-text>
          <n-text
            class="creator"
            @click="router.push(`/artist/songs?id=${albumDetail.artist.id}`)"
          >
            {{ albumDetail.artist.name }}
          </n-text>
        </div>
        <div class="intr">
          <span class="name">专辑简介</span>
          <span class="desc text-hidden">
            {{ albumDetail.description }}
          </span>
          <n-button
            class="all-desc"
            block
            strong
            secondary
            v-if="albumDetail?.description.length > 70"
            @click="albumDescShow = true"
          >
            全部简介
          </n-button>
          <n-modal
            class="s-modal"
            v-model:show="albumDescShow"
            preset="card"
            title="歌单简介"
            :bordered="false"
          >
            <n-scrollbar>
              <n-text v-html="albumDetail.description.replace(/\n/g, '<br>')" />
            </n-scrollbar>
          </n-modal>
        </div>
        <n-space class="tag" v-if="albumDetail.tags">
          <n-tag
            class="tags"
            round
            :bordered="false"
            v-for="item in albumDetail.tags"
            :key="item"
          >
            {{ item }}
          </n-tag>
        </n-space>
        <n-space class="control">
          <n-button strong secondary round type="primary" @click="playAllSong">
            <template #icon>
              <n-icon :component="MusicList" />
            </template>
            播放
          </n-button>
          <n-dropdown
            placement="right-start"
            trigger="click"
            :show-arrow="true"
            :options="dropdownOptions"
          >
            <n-button strong secondary circle>
              <template #icon>
                <n-icon :component="More" />
              </template>
            </n-button>
          </n-dropdown>
        </n-space>
      </div>
    </div>
    <div class="right">
      <div class="meta">
        <n-text class="name">{{ albumDetail.name }}</n-text>
        <n-text
          class="creator"
          @click="router.push(`/artist/songs?id=${albumDetail.artist.id}`)"
        >
          {{ albumDetail.artist.name }}
        </n-text>
        <div class="time">
          <div class="createTime">
            <span class="num">发行时间：</span>
            {{ getLongTime(albumDetail.publishTime) }}
          </div>
          <div class="company" v-if="albumDetail.company">
            <span class="num">发行公司：</span>
            {{ albumDetail.company }}
          </div>
        </div>
      </div>
      <DataLists :listData="albumData" hideAlbum />
    </div>
  </div>
  <div class="title" v-else-if="!albumId">
    <span class="key">参数不完整</span>
    <br />
    <n-button strong secondary @click="router.go(-1)" style="margin-top: 20px">
      返回上一级
    </n-button>
  </div>
  <div class="loading" v-else>
    <div class="left">
      <n-skeleton class="pic" />
      <n-skeleton text :repeat="5" />
      <n-skeleton text style="width: 60%" />
    </div>
    <div class="right">
      <n-skeleton :sharp="false" height="80px" width="60%" />
      <n-skeleton height="100%" width="100%" />
    </div>
  </div>
</template>

<script setup>
import { NIcon, NAvatar, NText } from "naive-ui";
import { getAlbum, likeAlbum } from "@/api/album";
import { useRouter } from "vue-router";
import { getSongTime, getLongTime } from "@/utils/timeTools";
import { MusicList, LinkTwo, More, Like, Unlike } from "@icon-park/vue-next";
import { userStore, musicStore, settingStore } from "@/store";
import DataLists from "@/components/DataList/DataLists.vue";

const router = useRouter();
const user = userStore();
const music = musicStore();
const setting = settingStore();

// 专辑数据
const albumId = ref(router.currentRoute.value.query.id);
const albumDetail = ref(null);
const albumData = ref([]);
const albumDescShow = ref(false);

// 图标渲染
const renderIcon = (icon) => {
  return () => {
    return h(
      NIcon,
      { style: { transform: "translateX(2px)" } },
      {
        default: () => icon,
      }
    );
  };
};

// 封面图像地址
const getCoverUrl = (url) => {
  const imageUrl = url.replace(/^http:/, "https:");
  if (imageUrl.endsWith(".jpg")) {
    return imageUrl + "?param=1024y1024";
  }
  return imageUrl;
};

// 判断收藏还是取消
const isLikeOrDislike = (id) => {
  const playlists = user.getUserAlbumLists.list;
  if (playlists.length) {
    return !playlists.some((item) => item.id === Number(id));
  }
  return true;
};

// 专辑下拉菜单数据
const dropdownOptions = ref([]);

// 更改专辑下拉菜单数据
const setDropdownOptions = () => {
  dropdownOptions.value = [
    {
      key: "copy",
      label: "复制专辑链接",
      props: {
        onClick: () => {
          if (navigator.clipboard) {
            try {
              navigator.clipboard.writeText(
                `https://music.163.com/#/playlist?id=${albumId.value}`
              );
              $message.success("专辑链接复制成功");
            } catch (err) {
              $message.error("复制失败：", err);
            }
          } else {
            $message.error("您的浏览器暂不支持该操作");
          }
        },
      },
      icon: renderIcon(h(LinkTwo)),
    },
    {
      key: "like",
      label: isLikeOrDislike(albumId.value) ? "收藏专辑" : "取消收藏专辑",
      show: user.userLogin,
      props: {
        onClick: () => {
          toChangeLike(albumId.value);
        },
      },
      icon: renderIcon(h(isLikeOrDislike(albumId.value) ? Like : Unlike)),
    },
  ];
};

// 获取歌单信息
const getAlbumData = (id) => {
  getAlbum(id).then((res) => {
    console.log(res);
    // 专辑信息
    albumDetail.value = res.album;
    $setSiteTitle(res.album.name + " - 专辑");
    // 专辑歌曲
    if (res.songs) {
      albumData.value = [];
      res.songs.forEach((v, i) => {
        albumData.value.push({
          id: v.id,
          num: i + 1,
          name: v.name,
          artist: v.ar,
          album: v.al,
          alia: v.alia,
          time: getSongTime(v.dt),
          fee: v.fee,
          sourceId: id,
          pc: v.pc ? v.pc : null,
          mv: v.mv ? v.mv : null,
        });
      });
    } else {
      $message.error("获取专辑歌曲失败");
    }
  });
};

// 播放专辑所有歌曲
const playAllSong = () => {
  try {
    // 获取元素
    const songDom = document.getElementById("datalists").firstElementChild;
    const allSongDom = document.querySelectorAll("#datalists > *");
    // 是否有元素存在 play
    let isHasPlay = false;
    // 遍历
    allSongDom.forEach((child) => {
      if (child.classList.contains("play")) {
        isHasPlay = true;
      }
    });
    if (!isHasPlay) {
      // 双击操作
      const event = new MouseEvent("dblclick", {
        bubbles: true,
        cancelable: true,
        view: window,
      });
      // 双击或单击
      if (setting.listClickMode === "dblclick") {
        songDom.dispatchEvent(event);
      } else if (setting.listClickMode === "click") {
        songDom.click();
      }
    } else {
      music.setPlayState(true);
    }
  } catch (err) {
    console.error("播放全部歌曲失败：" + err);
    $message.error("播放全部歌曲失败，请重试");
  }
};

// 收藏/取消收藏
const toChangeLike = async (id) => {
  const type = isLikeOrDislike(id) ? 1 : 2;
  try {
    const res = await likeAlbum(type, id);
    if (res.code === 200) {
      $message.success(`专辑${type == 1 ? "收藏成功" : "取消收藏成功"}`);
      user.setUserAlbumLists(() => {
        setDropdownOptions();
      });
    } else {
      $message.error(`专辑${type == 1 ? "收藏失败" : "取消收藏失败"}`);
    }
  } catch (err) {
    $message.error(`专辑${type == 1 ? "收藏失败" : "取消收藏失败"}`);
    console.error(`专辑${type == 1 ? "收藏失败：" : "取消收藏失败："}` + err);
  }
};

onMounted(() => {
  if (albumId.value) {
    getAlbumData(albumId.value);
    if (
      user.userLogin &&
      !user.getUserAlbumLists.has &&
      !user.getUserAlbumLists.isLoading
    ) {
      user.setUserAlbumLists(() => {
        setDropdownOptions();
      });
    } else {
      setDropdownOptions();
    }
  }
});

// 监听路由参数变化
watch(
  () => router.currentRoute.value,
  (val) => {
    albumId.value = val.query.id;
    if (val.name == "album") {
      getAlbumData(albumId.value);
    }
  }
);
</script>

<style lang="scss" scoped>
.playlist,
.loading {
  display: flex;
  flex-direction: row;
  .left {
    width: 40vw;
    height: 100%;
    max-width: 320px;
    min-width: 200px;
    margin-right: 20px;
    display: flex;
    flex-direction: column;
    align-items: flex-start;
    position: sticky;
    top: 24px;
    @media (max-width: 990px) {
      margin-right: 0;
      width: 30vw;
    }
    .cover {
      position: relative;
      display: flex;
      align-items: center;
      justify-content: flex-start;
      width: 80%;
      height: 80%;
      .n-avatar {
        border-radius: 8px;
        width: 100%;
        height: 100%;
      }
      .album {
        height: 100%;
        position: absolute;
        top: 0;
        right: -20%;
      }
    }
    .meta {
      width: 100%;
      height: 100%;
      display: flex;
      flex-direction: column;
      justify-content: flex-start;
      .title {
        display: none;
        flex-direction: column;
        margin-top: 0;
        .name {
          font-size: 28px;
          font-weight: bold;
          -webkit-line-clamp: 2;
        }
        .creator {
          margin-top: 6px;
          font-size: 16px;
          opacity: 0.8;
        }
      }
      .intr {
        margin-top: 24px;
        width: 80%;
        padding-left: 4px;
        .name {
          display: block;
          font-size: 20px;
          font-weight: bold;
          margin-bottom: 12px;
        }
        .desc {
          -webkit-line-clamp: 4;
          line-height: 26px;
          margin-bottom: 16px;
        }
      }
      .tag {
        margin-top: 20px;
        .tags {
          line-height: 0;
          font-size: 13px;
          cursor: pointer;
          transition: all 0.3s;
          &:hover {
            background-color: var(--main-second-color);
            color: var(--main-color);
          }
          &:active {
            transform: scale(0.95);
          }
        }
      }
      .control {
        margin-top: 20px;
      }
    }
  }
  .right {
    flex: 1;
    .meta {
      display: flex;
      flex-direction: column;
      margin-top: 20px;
      margin-bottom: 20px;
      .name {
        font-size: 30px;
        font-weight: bold;
      }
      .creator {
        margin-top: 6px;
        font-size: 16px;
        opacity: 0.8;
        cursor: pointer;
        transition: all 0.3s;
        &:hover {
          opacity: 1;
          color: var(--main-color);
        }
      }
      .time {
        margin-top: 8px;
        display: flex;
        flex-direction: row;
        align-items: center;
        @media (max-width: 1100px) {
          flex-direction: column;
          align-items: flex-start;
        }
        .num {
          color: #999;
        }
        div {
          margin-right: 12px;
        }
      }
    }
    .datalists {
      :deep(.songs) {
        @media (max-width: 990px) {
          .album,
          .time {
            display: none;
          }
        }
      }
    }
  }
  @media (max-width: 768px) {
    flex-direction: column;
    .left {
      position: relative;
      top: 0;
      width: 100%;
      height: 30vw;
      max-width: none;
      display: flex;
      flex-direction: row;
      .cover {
        height: 100%;
        min-width: 30vw;
        width: 30vw;
        margin-right: 60px;
      }
      .meta {
        .title {
          display: flex;
          margin-bottom: 16px;
          .name {
            font-size: 25px;
          }
          .creator {
            font-size: 14px;
          }
        }
        .intr {
          margin-top: 0;
          padding-left: 0;
          .name,
          .all-desc {
            display: none;
          }
          .desc {
            -webkit-line-clamp: 3;
            margin-bottom: 0;
          }
        }
        .control {
          position: absolute;
          left: 0;
          bottom: -60px;
        }
      }
    }
    .right {
      margin-top: 80px;
      .meta {
        display: none;
      }
    }
  }
  @media (max-width: 540px) {
    .left {
      .cover {
        margin-right: 44px;
      }
      .meta {
        .title {
          margin-bottom: 0;
          .name {
            font-size: 24px;
          }
        }
        .intr,
        .tag {
          display: none !important;
        }
        .control {
          position: static;
        }
      }
    }
    .right {
      margin-top: 30px;
    }
  }
  @media (max-width: 520px) {
    .left {
      .meta {
        .title {
          .name {
            font-size: 20px;
          }
          .creator {
            font-size: 12px;
          }
        }
      }
    }
  }
  @media (max-width: 370px) {
    .left {
      .meta {
        .title {
          .name {
            -webkit-line-clamp: 3;
          }
        }
        .control {
          position: absolute;
        }
      }
    }
    .right {
      margin-top: 80px;
    }
  }
}
.title {
  margin-top: 30px;
  margin-bottom: 20px;
  font-size: 24px;
  .key {
    font-size: 40px;
    font-weight: bold;
    margin-right: 8px;
  }
}
.loading {
  .left {
    display: block;
    .pic {
      padding-bottom: 100%;
      width: 100%;
      height: 0;
      border-radius: 8px !important;
      margin-bottom: 20px;
    }
  }
  .right {
    .n-skeleton {
      margin-bottom: 20px;
    }
  }
}
</style>
