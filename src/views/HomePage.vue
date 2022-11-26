<template>
  <ion-page>
    <ion-header :translucent="true">
      <ion-toolbar>
        <ion-title>
          <div class="flex-row-center space-between">
            <div>Music Player</div>
            <button @click="isOpen = true">Playlist</button>
          </div>
        </ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">Songs</ion-title>
        </ion-toolbar>
      </ion-header>

      <div id="container">
        <div class="boom-box">
          <div
            :class="['boom-track-circle', selectedSong?.isPlaying && 'grad']"
          ></div>
        </div>
        <div v-if="selectedSong">
          <div>
            <h2>{{ selectedSong?.artist }}</h2>
            <p>{{ selectedSong?.songTitle }}</p>
          </div>
          <div class="pad-top">
            <button @click="changeSong(-1)">Prev</button>
            <button
              v-if="selectedSong.isPlaying"
              @click="pauseSong(selectedSong?.idx)"
            >
              Pause
            </button>
            <button v-else @click="playSong(selectedSong?.idx)">Play</button>
            <button @click="changeSong(1)">Next</button>
          </div>
        </div>
        <div v-else>
          <h2>No Song is selected Now!!!</h2>
          <button @click="selectSongFromList">Play Random Song</button>
        </div>
        <div class="pad-top">
          <input type="file" ref="file" @change="changeFile" />
        </div>
      </div>

      <ion-modal :is-open="isOpen">
        <ion-header>
          <ion-toolbar>
            <ion-title>Song List</ion-title>
            <ion-buttons slot="end">
              <ion-button @click="isOpen = false">Close</ion-button>
            </ion-buttons>
          </ion-toolbar>
        </ion-header>
        <ion-content class="ion-padding">
          <template v-for="song in songList" :key="song.idx">
            <div class="flex-row-center space-between tab">
              <div>
                <p class="header">{{ song.artist }}</p>
                <p>{{ song.songTitle }}</p>
              </div>
              <div>
                <button v-if="song.isPlaying" @click="pauseSong(song.idx)">
                  Pause
                </button>
                <button v-else @click="playSong(song.idx)">Play</button>
              </div>
            </div>
          </template>
        </ion-content>
      </ion-modal>

    </ion-content>
  </ion-page>
</template>

<script lang="ts" setup>
import {
  IonContent,
  IonModal,
  IonHeader,
  IonPage,
  IonTitle,
  IonToolbar,
} from "@ionic/vue";

import { Directory, Filesystem, ReadFileOptions } from "@capacitor/filesystem";
import { computed, onBeforeMount, ref } from "vue";

const isOpen = ref(false);
const audioplayer = ref<HTMLAudioElement>(new Audio());
const songList = ref<
  Array<{
    songTitle: string;
    isPlaying: boolean;
    selected: boolean;
    artist: string;
    idx: number;
  }>
>([]);

const selectedSong = computed(() => {
  return songList.value.find((e) => e.selected);
});

onBeforeMount(async () => {
  readMusicFiles();

  audioplayer.value.onended = () => {
    changeSong(1);
  };
});

function blobToBase64(blob: Blob): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.onerror = reject;
    reader.onload = (e: any) => {
      resolve(e.target.result as any);
    };
    reader.readAsDataURL(blob);
  });
}

async function changeFile(evnt: any) {
  const blobfile = evnt.target.files[0];
  if (!blobfile) return;

  const data = await blobToBase64(blobfile);
  await Filesystem.writeFile({
    path: `Music/${blobfile.name}`,
    data: data,
    directory: Directory.Documents,
  });

  readMusicFiles();
}

async function readMusicFiles() {
  try {
    await Filesystem.mkdir({
      path: "Music",
      directory: Directory.Documents,
    });
  } catch (err) {
    const dir = await Filesystem.readdir({
      path: "Music",
      directory: Directory.Documents,
    });

    songList.value = dir.files.map((e, idx) => {
      return {
        idx,
        artist: e.name.replace(".mp3", ""),
        songTitle: e.name,
        ...e,
        isPlaying: false,
        selected: false,
      };
    });

    resetList();
    audioplayer.value.pause();
  }
}

function selectSongFromList() {
  if (songList.value.length) {
    songList.value[0].selected = true;
  }
}

function resetList() {
  songList.value.forEach((e) => {
    e.isPlaying = false;
    e.selected = false;
  });
}

function getSongLocationOnLoadedList(idx: number) {
  const songIdx = songList.value.findIndex((e) => e.idx === idx);
  if (songIdx === -1) {
    alert("song not found");
    return null;
  }
  return songIdx;
}

const readFile = async (params: ReadFileOptions) => {
  const file = await Filesystem.readFile(params);
  const str = file.data.replace("data:application/octet-stream;base64,", "");
  return `data:audio/mpeg;base64,${str}`;
};

async function playSong(idx: any) {
  audioplayer.value.pause();

  const index = getSongLocationOnLoadedList(idx);
  if (index === null) return;

  const data = await readFile({
    path: `Music/${songList.value[index].songTitle}`,
    directory: Directory.Documents,
  });

  resetList();

  songList.value[index].isPlaying = true;
  songList.value[index].selected = true;

  audioplayer.value.src = data;
  audioplayer.value.play();
}

function pauseSong(idx: any) {
  const index = getSongLocationOnLoadedList(idx);
  if (index === null) return;
  resetList();

  songList.value[index].isPlaying = false;
  songList.value[index].selected = true;
  audioplayer.value?.pause();
}

function changeSong(incValue: number) {
  if (!selectedSong.value) {
    selectSongFromList();
    return;
  }

  const idx = selectedSong.value.idx + incValue;

  if (idx < 0 || idx >= songList.value.length) {
    alert("no more songs left");
    return;
  }
  if (selectedSong.value.isPlaying) {
    playSong(idx);
    return;
  }
  resetList();
  const song = songList.value.find((e) => e.idx === idx);
  if (!song) return;

  song.isPlaying = false;
  song.selected = true;
}
</script>

<style scoped>
#container {
  text-align: center;
  position: absolute;
  left: 0;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
  padding-left: 1rem;
  padding-right: 1rem;
}

#container p {
  font-size: 16px;
  line-height: 22px;
  color: #8c8c8c;
  margin: 0;
}

p,
.header {
  margin: 0 !important;
}
p.header {
  font-weight: bold;
}

button {
  padding: 1rem;
  font-weight: bold;
}

button:hover {
  background-color: #8c8c8c;
}

.tab {
  border-top: 1px solid rgba(100, 100, 100, 0.461);
  padding: 1rem;
}

.boom-box {
  margin: 1rem auto;
  max-width: 30rem;
  display: flex;
  align-items: center;
  justify-content: center;
  aspect-ratio: 1/1;
  box-shadow: 0px 0px 3px rgba(0, 0, 0, 0.3);
}

.boom-track-circle {
  width: 70%;
  aspect-ratio: 1/1;
  border-radius: 50%;
  /* background: rgb(2,0,36); */
  background: radial-gradient(
    circle,
    rgba(2, 0, 36, 1) 75%,
    rgba(9, 9, 121, 1) 92%,
    rgba(0, 212, 255, 1) 100%
  );
  position: relative;
  box-shadow: 0px 0px 3px rgba(231, 231, 231, 0.5);
}

.boom-track-circle::after {
  content: " ";
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 20%;
  aspect-ratio: 1/1;
  box-shadow: 0px 0px 3px rgba(231, 231, 231, 0.5);
  border-radius: 50%;
  background-color: aliceblue;
}

.flex-row-center {
  display: flex;
  flex-direction: row;
  align-items: center;
}
.space-between {
  justify-content: space-between;
}
.pad-top {
  padding-top: 2rem;
}
.grad {
  background: linear-gradient(
    -50deg,
    #fa3f06,
    #a1aa44,
    #1b4251,
    #23d5ab
  ) !important;
  background-size: 400% 400%;
  animation: gradient 10s ease infinite;
}

@keyframes gradient {
  0% {
    background-position: 0% 50%;
    transform: rotate(0deg);
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
    transform: rotate(360deg);
  }
}

</style>
